# Data and Validation

This document defines how Labrodev projects handle input mapping and validation.

There are no Request classes in this architecture. All input mapping and validation is handled exclusively by Data classes using Spatie Laravel Data.

Validation exists only for write operations (store and update). Read-side operations do not use validation or Data objects.

Casting is a first-class mechanism in this system. We use both:
- Spatie Data casts (for mapping input into Data attributes)
- Custom Cast classes (domain-oriented casting and normalization), which may be reused by Spatie Data casts or used internally by the domain

---

## Core principles

- Data classes are the single source of truth for input contracts.
- Controllers do not validate and do not use Request classes.
- Validation is applied only for store and update operations.
- Read-side operations do not use validation or Data objects.
- Domain code assumes it receives valid, already-mapped Data objects.
- Arrays are not input contracts. Data objects are.

---

## Data objects as the write-side boundary

Data classes exist only to support write-side operations.

They are used to:
- define allowed input fields
- validate input for store/update
- cast raw input into typed objects
- provide a clear, typed contract for Actions and Orchestrators

Controllers receive Data objects via method injection — Laravel and Spatie Data handle the mapping automatically.

A controller must **type-hint the Data class directly** in its `__invoke()` signature:

```php
public function __invoke(MyData $myData, MyAction $myAction): RedirectResponse
```

A controller must **never**:
- inject `Illuminate\Http\Request` to manually construct a Data object
- call `MyData::from($request->all())` or `MyData::from([...])` with manual field mapping
- use `$request->input()` to pluck fields into a Data object

Spatie Data automatically resolves the Data class from the request when type-hinted as a controller parameter. Manual construction is redundant, error-prone, and violates this architecture.

Domain code must never depend on how Data objects are created.

---

## Scope of validation

Validation exists only for:

- store (create)
- update

Rules:
- all validation rules live inside Data classes
- controllers must not perform validation
- Core Actions and Orchestrators assume valid Data objects

Validation does not exist for:
- index queries
- filtering
- sorting
- pagination
- exports

Read-side input is treated as tolerant input and must be handled defensively in queries, not validated.

---

## What validation is allowed to do

Validation rules may enforce:
- required or nullable fields
- primitive types and formats
- enum constraints
- existence checks
- value ranges

Validation must not:
- implement business rules
- depend on current domain state beyond simple existence
- decide whether an operation is allowed

Business decisions belong to Core logic, not validation.

---

## Casting: Spatie Data casts and custom Casts

Casting is used to transform validated primitives into richer, typed attributes.

We distinguish two kinds of casting:

### Spatie Data casts

Spatie Data casts are used at the delivery boundary to map input into Data attributes.

They answer: "How do we build this typed Data attribute from raw input values?"

Typical use cases:
- casting UUID strings into Model instances
- casting strings into Enums
- casting arrays into Data objects

### Custom Cast classes

Custom Cast classes are part of the Core and express domain-specific casting and normalization.

They answer: "How do we consistently convert between raw/persisted representations and domain-friendly values?"

Typical use cases:
- UUID or identifier normalization
- value object parsing/formatting
- consistent domain conversions reused across:
    - Data classes
    - Eloquent casts
    - internal domain logic

Custom Casts must remain free of delivery concerns. They must not depend on controllers, HTTP, or UI.

---

## Relations in Data classes use UUID casters — never internal IDs

Data classes must never expose internal database IDs (`_id` fields).

When a Data class refers to a related model, it must:
- accept a UUID string as input under the **entity name** key (e.g. `service`, not `service_id` or `service_uuid`)
- declare a typed model property with the same name: `public ?Service $service = null`
- apply a `#[WithCast(ServiceUuidCaster::class)]` attribute to resolve the UUID into a model
- validate the raw input key as `exists:<table>,uuid`

Required pattern:
```php
#[WithCast(ServiceUuidCaster::class)]
public ?Service $service = null,
```

Validation:
```php
'service' => ['required', 'string', 'uuid', 'exists:services,uuid'],
```

Forbidden:
- `public ?int $service_id = null` — leaks internal database IDs
- `public ?string $service_uuid = null` — raw UUID string without casting
- Any `_id` suffixed field that references a related model

The UUID caster lives in the model's domain: `Core\Domain\{Domain}\Casts\{Model}UuidCaster`.

---

## Computed fields do not belong in Data classes

Data classes represent **input contracts** — what the client sends.

Fields that are derived, calculated, or resolved by the backend must not appear in Data classes:
- service time (calculated from service + parameters)
- price (calculated from service + configuration)
- status (set by the action)
- timestamps

If a field is never submitted by the client, it does not belong in the Data class.

---

## Validation always targets raw input keys

Important rule:
- validation always applies to the raw input key/value
- casting happens after validation
- typed Data attributes must have their source keys validated

Example scenario:
- incoming input provides `service` (UUID string)
- Data class exposes `Service $service` (typed model)
- a UUID caster converts the validated UUID into a Service instance

In this case:
- validation rules must validate `service`:
    - required
    - string
    - uuid
    - exists:services,uuid
- casting then converts the validated UUID into a Service instance

Casting does not replace validation.

---

## Data class responsibilities

A Data class may:
- define validation rules
- normalize values (trim strings, type normalization)
- use Spatie Data casts and custom Casts to resolve typed attributes

A Data class must not:
- use the `readonly` class modifier — `Spatie\LaravelData\Data` is not readonly, so a readonly subclass causes a fatal error in PHP 8.2+
- implement business logic
- perform authorization decisions
- mutate domain state
- contain workflows

---

## Error handling

Validation failures are delivery-layer concerns.

Rules:
- validation errors must not be expressed as domain exceptions
- validation failures stop execution before Core logic
- domain exceptions represent business failures, not invalid input

---

## Naming and attributes

Data class attributes follow these rules:

- all attributes use snake_case
- relation attributes use the **entity name** (e.g. `$service`, `$postal_code`), never `_id` or `_uuid` suffixes
- scalar attributes match incoming input keys exactly (e.g. `$square_meters`, `$discount_code`)
- typed properties may represent richer objects after casting

Consistency between input keys, validation rules, and casts is mandatory.

---

## Summary

- No Request classes
- No read-side Data classes
- No validation outside store/update
- All validation lives in Data classes
- Casting is mandatory and uses both Spatie Data casts and custom Cast classes
- Domain code never sees raw input