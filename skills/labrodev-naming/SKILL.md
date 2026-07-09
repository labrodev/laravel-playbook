---
name: labrodev-naming
description: "Use when naming anything in a Labrodev Laravel project — domains, classes (Actions, Services, Controllers, Data, Queries, Policies, Enums, ...), methods, variables — and whenever invoking an Action/Service or declaring typed parameters. Governs the class-suffix pattern table, the named-arguments callable invocation contract, and the typed-parameter mirror rule (BookingData $bookingData, never $data)."
license: MIT
metadata:
  author: labrodev
---

# Labrodev Naming & Named Arguments

Part of the Labrodev playbook skill set — this is one of the two foundation skills (with labrodev-core) that every other labrodev-* skill assumes. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

Naming is treated as architecture. Consistent naming reduces cognitive load, makes code searchable, and makes AI-assisted generation reliable. These rules are defaults: if a domain has strong ubiquitous language that conflicts with a rule, the domain language wins — but apply the decision consistently.

## Musts

- Everything is **singular**: domain folders, namespaces, classes. `BookingController`, `BookingCollection`, `Core/Domain/Booking` — never `BookingsController`, `Core/Domain/Bookings`.
- Names must be **distinctive and self-explanatory** — a name must communicate responsibility without reading the code.
- **Actions**: `Model + VerbInBaseForm`, entity first — `BookingCreate`, `BookingUpdate`, `BookingCancel`. Single public entry point `__invoke()`.
- **Services**: `Noun + VerbAgent` — `EmailSender`, `PriceCalculator`, `TokenGenerator`. Single-operation services expose `__invoke()`; multi-step services may use explicit named methods.
- **Jobs**: `Verb + Object + Job` — `SendBookingConfirmationJob`. Jobs are thin wrappers delegating to Core Actions.
- **Controllers**: `{Model}{Action}Controller` — `BookingStoreController`, `BookingIndexController`, `BookingUpdateController`. One per action, invokable.
- **Most domain components are prefixed with the entity name**: `BookingRule`, `BookingPolicy`, `BookingObserver`, `BookingUuidCaster`, `BookingCollection`, `BookingData`, `BookingConfirmedEvent`. Exceptions: Models (`Booking`) and Actions (`BookingCreate` — verb pattern, no suffix).
- **Enums**: concept noun, no `Enum` suffix/prefix — `BookingStatus`, not `BookingStatusEnum`.
- **Methods** are verbal and explicit: `create()`, `fetchBooking()`, `calculatePrice()`, `updateBookingQuantity()`.
- **Query methods** follow the grammar `by{X}` for filtering by an attribute/identifier and `for{X}` for scoping to a related entity: `byId(int $id)`, `byUuid(string $uuid)`, `byUuids(array $uuids)`, `byStatus(BookingStatus $bookingStatus)`, `forCustomer(Customer $customer)`.
- **Variables**: camelCase, explicit and semantic — `$customerEmail`, `$totalPrice`, `$bookingStartTimestamp`.
- **Model and Data class attributes**: snake_case — `$booking_id`, `$customer_id` — because they map to database columns and external representations.
- **Typed-parameter mirror rule**: a parameter, closure parameter, or local variable whose type is a concrete named class (`*Data`, envelope, DTO, named domain type) MUST be named as the **class short name in camelCase**: `BookingData $bookingData`, `CustomerAddressData $customerAddressData`.
- **Named arguments**: any call with **more than one argument** uses named arguments, kept in a consistent order (alphabetical if no stronger local convention). Single-argument calls may stay positional.
- **Actions and Services are invoked as callables**: injected via `__invoke()` parameters and called as `$bookingCreate(bookingData: $bookingData)`.
- Required domain dependencies stay **non-nullable** in signatures: `Booking $booking`, not `?Booking $booking`, unless the plan models absence explicitly.

## Must-nots

- No vague class names: `Manager`, `Handler`, `Processor`, `Util`, bare `Service`, `Helper` (except in `Core/Support/Helpers` where it is explicit).
- No verb-first Actions: `CreateBooking` is wrong; `BookingCreate` is right.
- No generic-verb Actions: `Handle`, `Process`, `ExecuteAction`.
- No `->execute()` / `->handle()` on Actions or Services — they are callables via `__invoke()`. Exceptions: `Job::handle()`, `Pipeline::handle()` (framework/pattern contracts).
- No extra public entry methods on Actions — `__invoke()` only; private helpers are fine.
- No method names that hide intent: `handle()`, `process()`, `do()`, `run()`, `make()`.
- Never `$data`, `$dto`, `$payload` (as a generic name), `$addr`, `$b`, `$m` for typed value objects — the variable name mirrors the class short name.
- No abbreviated or vague variables: `$cursor`, `$bStart`, `$prevTs`, `$a`/`$b` in comparators.
- No positional arguments on multi-argument calls: `$bookingCreate($customer, $bookingData)` is forbidden.
- No plural anywhere in class or domain names.

## Naming reference table

| Component | Pattern | Example (Booking domain) |
|---|---|---|
| Domain folder / namespace | singular business noun | `Core/Domain/Booking`, `App/Layer/Dashboard/Booking` |
| Model | entity noun, no suffix | `Booking` |
| Action | `{Model}{Verb}` (verb in base form, entity first) | `BookingCreate`, `BookingCancel` |
| Service | `{Noun}{VerbAgent}` | `BookingReminderSender`, `PriceCalculator` |
| Job | `{Verb}{Object}Job` | `SendBookingConfirmationJob` |
| Inertia Controller | `{Model}{Action}Controller` | `BookingStoreController`, `BookingIndexController` |
| JsonController | `{Model}{Verb}Controller` | `BookingSearchController` |
| Data | `{Model}Data` — default, shared by create + update; `{Model}CreateData`/`{Model}UpdateData` only when fields differ (→ labrodev-data skill) | `BookingData` |
| Core Query | `{Model}Query` (one per Model) | `BookingQuery` |
| Layer IndexQuery | `{Model}IndexQuery` | `BookingIndexQuery` |
| ViewModel | `{Model}{Purpose}ViewModel` | `BookingIndexViewModel`, `BookingShowViewModel` |
| Resource | `{Model}Resource` | `BookingResource` |
| Policy | `{Model}Policy` | `BookingPolicy` |
| Rule | `{Model}Rule` (exactly one per Model) | `BookingRule` |
| Observer | `{Model}Observer` | `BookingObserver` |
| Collection | `{Model}Collection` | `BookingCollection` |
| Caster | `{Model}UuidCaster`, `{Model}CollectionCaster` | `BookingUuidCaster` |
| Enum | concept noun, no `Enum` affix | `BookingStatus` |
| Event | `{Entity}{PastTenseFact}Event` — names a fact, never a command | `BookingConfirmedEvent` (not `ConfirmBookingEvent`) |
| Exception | `{DescriptiveCondition}Exception` (placement + `::make()` contract → labrodev-exception skill) | `BookingOverlapException` |
| Pipeline-orchestrating Service | `{Workflow}Service` in `Services/` (→ labrodev-pipeline skill) | `BookingRegistrationService` |
| Payload | `{Entity}{Process}Payload` | `BookingEvaluationPayload` |
| Pipeline step | verb-first atomic step in `Pipelines/{Workflow}/`, no `Pipeline` suffix | `ConfirmBooking`, `RecalculateBookingTotals` |

For the anatomy and responsibilities of each component: → see the labrodev-controller, labrodev-viewmodel-resource, labrodev-query, labrodev-data, labrodev-model, labrodev-action, labrodev-authorization, and labrodev-enum skills. This skill owns only their names and how they are invoked.

## The invocation contract (canonical template)

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

For everything inside the Action body (transactions, UUID assignment, Rule guards, failure modes): → see the labrodev-action skill.
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
- **Legacy zone**: vendor/starter code under `app/Http`, `app/Models`, `app/Actions/Fortify` is exempt → see the labrodev-core skill.

## Review checklist

- Are all folders, domains, and class names singular?
- Do Actions follow `{Model}{Verb}` (entity first) with `__invoke()` as the only public entry point?
- Does every controller follow `{Model}{Action}Controller` and every prefixed component carry its entity prefix (`BookingRule`, `BookingPolicy`, ...)?
- Are Actions/Services invoked as callables — no `->execute()` / `->handle()` outside Job/Pipeline?
- Do all multi-argument calls use named arguments in a consistent order?
- Does every typed value-object parameter and local variable mirror its class short name in camelCase (`BookingData $bookingData`, never `$data`)?
- Are Model/Data attributes snake_case and everything else camelCase?
- Are there zero vague names (`Manager`, `Handler`, `Processor`, `handle()`, `process()`, `run()`)?
- Do query methods follow the `by{X}` / `for{X}` grammar?
- Are enums free of the `Enum` affix and events named as past-tense facts?
