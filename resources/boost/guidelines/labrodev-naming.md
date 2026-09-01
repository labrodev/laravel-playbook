# Labrodev Naming & Named Arguments

Always-on law: naming is architecture. If a domain's ubiquitous language conflicts with a rule, the domain language wins — applied consistently. Worked examples and edge cases → the `labrodev-naming` skill.

## Musts

- Everything is **singular**: domain folders, namespaces, classes (`Core/Domain/Booking`, `BookingController` — never `Bookings`).
- Names are distinctive and self-explanatory — responsibility readable without opening the class.
- Methods are verbal and explicit (`create()`, `fetchBooking()`, `calculatePrice()`); query methods follow `by{X}` (attribute filter) / `for{X}` (related-entity scope).
- Variables are camelCase and semantic (`$customerEmail`, `$totalPrice`); Model and Data class attributes are snake_case (they map to columns and external representations).
- **Typed-parameter mirror rule**: any parameter, closure parameter, or local variable of a concrete named type is named as the class short name in camelCase — `BookingData $bookingData`, never `$data`.
- **Named arguments** on every call with more than one argument, in a consistent (default alphabetical) order. Single-argument calls may stay positional.
- **Actions and Services are callables**: injected via `__invoke()` parameters and invoked as `$bookingCreate(bookingData: $bookingData);` — never `->execute()` / `->handle()` (only `Job::handle()` / `Pipeline::handle()` keep their framework entry points).
- Required domain dependencies stay non-nullable (`Booking $booking`, not `?Booking $booking`).

## Naming table

| Component | Pattern | Example |
|---|---|---|
| Domain folder / namespace | singular business noun | `Core/Domain/Booking` |
| Model | entity noun, no suffix | `Booking` |
| Action | `{Model}[{Aspect}]{Create\|Update\|Remove}` — entity first, always ending in one of the three verbs | `BookingCreate`, `BookingStatusUpdate` |
| Service | `{Noun}{-er agent}` — ends in `-er`, naming the manipulation inside | `PriceCalculator`, `EmailSender`, `BookingSearcher` |
| Job | `{Verb}{Object}Job` | `SendBookingConfirmationJob` |
| Inertia Controller | `{Model}{Action}Controller` | `BookingStoreController` |
| JsonController | `{Model}{Verb}Controller` | `BookingSearchController` |
| Data | `{Model}Data` shared by create+update; split `{Model}CreateData`/`{Model}UpdateData` only when fields differ | `BookingData` |
| Core Query | `{Model}Query` (one per Model) | `BookingQuery` |
| Layer IndexQuery | `{Model}IndexQuery` | `BookingIndexQuery` |
| ViewModel | `{Model}{Purpose}ViewModel` | `BookingIndexViewModel` |
| Resource | `{Model}Resource` | `BookingResource` |
| Policy / Rule / Observer / Collection | `{Model}` + suffix (`Rule`: exactly one per Model; Policy/Observer/Collection: mandatory trio per Model → labrodev-model) | `BookingPolicy`, `BookingRule` |
| Rule gate method | `can{Verb}` mirroring the operation (invariant checks: descriptive) | `canCreate`, `canUpdate`, `canRemove` |
| Infrastructure boundary DTO | `{Name}Payload` (or `{Name}Envelope` with transport metadata) | `OutboundMessagePayload` |
| Caster | `{Model}{Kind}Caster` | `BookingUuidCaster` |
| Enum | concept noun, no `Enum` affix | `BookingStatus` |
| Event | `{Entity}{PastTenseFact}Event` — a fact, never a command | `BookingConfirmedEvent` |
| Exception | `{DescriptiveCondition}Exception` | `BookingOverlapException` |
| Pipeline-orchestrating Service | `{Workflow}Service` in `Services/` | `BookingRegistrationService` |
| Payload | `{Entity}{Process}Payload` | `BookingEvaluationPayload` |
| Pipeline step | verb-first, no suffix, in `Pipelines/{Workflow}/` | `ConfirmBooking` |

## Must-nots

- No vague class names: `Manager`, `Handler`, `Processor`, `Util`, bare `Service`, `Helper` (except `Core/Support/Helpers`).
- No verb-first Actions (`CreateBooking`) and no generic-verb Actions (`Handle`, `Process`).
- No intent-hiding method names: `handle()`, `process()`, `do()`, `run()`, `make()` (exceptions: exception `::make()` factories → labrodev-exception; Payload `::make()` constructors → labrodev-pipeline).
- No `$data`, `$dto`, `$payload`-as-generic, or abbreviated variables (`$addr`, `$b`) for typed value objects.
- No positional arguments on multi-argument calls.
- No plural anywhere in class or domain names.

## The invocation contract

```php
public function __invoke(
    BookingData $bookingData,     // mirrors class short name — never $data
    BookingUpdate $bookingUpdate, // mirrors class short name — never $action
): RedirectResponse {
    $bookingUpdate(               // callable — never ->execute()/->handle()
        booking: $booking,
        bookingData: $bookingData,
    );
    // ...
}
```
