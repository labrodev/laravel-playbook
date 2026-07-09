# Database and Models

This document defines how Labrodev projects design database models and Eloquent usage inside Core/Domain.

Models represent persistence and database records. They are not the domain itself, but they must be predictable, explicit, and consistent. Business workflows must live in Actions (or a pipeline-orchestrating Service for staged workflows), not in Models.

---

## Core principles

- Models are persistence objects, not workflow coordinators.
- Every model extends the base model from Core/Shared.
- Mass assignment is not used. We do not use `$fillable`.
- Attributes are assigned explicitly (row by row) inside Actions.
- Relationships are always defined explicitly with correct relation types.
- Casting and visibility are always explicit.

---

## Base model

All models must extend the shared base model.

Rule:
- `Core/Domain/{Domain}/Models/{Model}.php` extends `Core/Shared/Models/BaseModel` (or your shared base class name)

This ensures consistent defaults and shared behavior across domains.

---

## Traits

Models use traits for shared, cross-cutting persistence logic.

Traits are the preferred mechanism for generic model concerns such as:
- actor/audit metadata (ModelHasActor)
- soft deletion (SoftDeletes)
- timestamps and common helpers when required

UUIDs are NOT a trait concern: they are assigned explicitly in the create Action
(`$model->uuid = (string) Str::uuid();`) — never in the model, boot(), or a trait.

Rule:
- Traits must contain persistence-level logic only.
- Traits must not contain business workflows or cross-entity coordination.
- Traits should be reusable and domain-agnostic; if a trait becomes domain-specific, it should live in the domain and be named accordingly.

---

## Table and columns

Rules:
- Use clear, singular model names and consistent table names.
- Declare the database table with **`#[Table('actual_table')]`** on the model class (`Illuminate\Database\Eloquent\Attributes\Table`). Do **not** use `protected $table`. (The model stub uses placeholder `{table}`; set it to the migration table name.)
- Prefer UUIDs where the domain requires stable external identifiers.
- Keep schema predictable: avoid implicit behavior and magic columns.

---

## Relationships

All relationships must be defined explicitly using correct Eloquent relation types.

Examples of required explicitness:
- `belongsTo`
- `hasOne`
- `hasMany`
- `belongsToMany`
- `morphOne`, `morphMany`, `morphTo`, etc. (when used)

Rules:
- Relationship methods must return the correct Relation type.
- Relationship names should be meaningful and singular/plural correctly.
- Avoid hidden relationship logic; keep it declarative.

---

## Route parameters (UUID)

Labrodev projects resolve models from URLs by **UUID**, not numeric `id`.

Rule:
- **Do not** add `getRouteKeyName()` on domain models for this. Keep the default route key on the model; binding is declared on the route.
- In route definitions, use Laravel’s explicit binding field: `{parameterName:uuid}` (column name after the colon).

Examples (`routes/supplier.php` or equivalent):

```php
Route::get('{booking:uuid}', BookingShowController::class)->name('show');
Route::get('{booking:uuid}/edit', BookingEditController::class)->name('edit');
Route::put('{booking:uuid}', BookingUpdateController::class)->name('update');

Route::prefix('{service:uuid}')->group(function () {
    Route::get('general', ...);
});
```

Effects:
- Incoming requests resolve the model with `where('uuid', $segment)`.
- `route('name', $model)` still substitutes the UUID when the named route uses `{…:uuid}`.

Every new resource that appears in the URL must use `:uuid` on those segments consistently.

---

## Invokable controllers and authorization (Laravel 13)

For single-action (`__invoke`) layer controllers (`App/Layer/Dashboard`, `App/Layer/Api`, etc.), **declare authorization on the controller class** with **`#[Authorize(...)]`** — do not call `$this->authorize(...)` in `__invoke()` unless an exception applies (see below).

```php
use Illuminate\Routing\Attributes\Controllers\Authorize;

#[Authorize(ProductPolicy::PERMISSION_CREATE, Product::class)]
final class ProductCreateController extends BaseController
{
    public function __invoke(): Response { ... }
}
```

For a route-model-bound instance, pass the **route parameter name** as the second argument (must match the route segment), e.g. `#[Authorize('update', 'service')]`.

**Policy abilities** (names, prefixes, Spatie permission keys) are project-specific. Document them for each layer; align `#[Authorize(...)]` with policies and domain `*Rule` checks. See `docs/01-project-structure.md` § Controllers.

**Keep `$this->authorize` in the controller method** when the authorized model is resolved only inside the action (e.g. `ConfigurationSet` after `ConfigurationSetFetcher`, or `Supplier` after `resolveSupplier()`). See `docs/01-project-structure.md` § Controllers.

## Collections: always define a custom collection

Every model must have a corresponding Collection class in the domain Collections folder.

Rule:
- For each model `{Model}`, define `{Model}Collection` in:
    - `Core/Domain/{Domain}/Collections/{Model}Collection.php`

The model must declare it with Laravel’s **`#[CollectedBy({Model}Collection::class)]`** attribute on the model class.

Do **not** override `newCollection()` for this — the attribute is the single source of truth (Laravel 13+).

This ensures:
- typed collections
- a consistent place for collection-specific logic
- predictable return types from queries

---

## Eloquent class attributes (Laravel 13+)

Domain models declare observer, collection, policy, and (when present) factory using **PHP attributes** on the class. Order is conventional:

```php
use Illuminate\Database\Eloquent\Attributes\CollectedBy;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;
use Illuminate\Database\Eloquent\Attributes\UseFactory;
use Illuminate\Database\Eloquent\Attributes\UsePolicy;

#[ObservedBy(ProductObserver::class)]
#[CollectedBy(ProductCollection::class)]
#[UsePolicy(ProductPolicy::class)]
#[UseFactory(ProductFactory::class)]  // omit until a factory exists (e.g. Draft)
final class Product extends BaseModel
```

Rules:

| Attribute | Purpose |
|-----------|---------|
| `#[ObservedBy]` | Registers the model observer (replaces `Model::observe()` and **must not** be called from `boot()` — that causes recursive boot). |
| `#[CollectedBy]` | Custom Eloquent collection for query results. |
| `#[UsePolicy]` | Policy class for `Gate` / authorization (replaces `fetchPolicyClass()`). |
| `#[UseFactory]` | Factory class for `Model::factory()` (requires `HasFactory` on `BaseModel`). |

- **Domain Rules** (`{Model}Rule`) hold object-state gates (`isEditable`, `canBeDeleted`, …). **Policies** import and call them directly, e.g. `{Model}Rule::isEditable($model)`. Do **not** add `fetchRuleClass()` on models.
- **Base model** must use **`HasFactory`** so `#[UseFactory]` works.
- Never register observers inside `static::boot()` via `observe()` on the same model.

---

## Observers

When a model has an observer, wire it with **`#[ObservedBy({Model}Observer::class)]`** on the model class.

Rules:
- Observer classes live in:
    - `Core/Domain/{Domain}/Observers/{Model}Observer.php`
- Do **not** register observers in a service provider for this pattern unless you have an exceptional case
- Observers must not implement workflows or business processes; they may enforce invariants and persistence synchronization only

If logic becomes workflow-like, it belongs in Actions (or a pipeline-orchestrating Service for staged workflows), not in Observers.

---

## No mass assignment, no fillable

We do not use `$fillable`.

Reasons:
- Actions assign attributes explicitly
- explicit assignment makes intent clear and reduces unexpected writes
- it prevents accidental overposting

Rules:
- Do not define `$fillable`
- Prefer explicit assignments like:
    - `$product->name = $data->name;`
    - `$product->status = $data->status;`
- If many attributes must be assigned, still prefer explicit assignment rather than `$model->fill()`.

---

## Factories

Factories are optional, but should exist when the model is used in tests or requires consistent seeding.

Rules:
- Define a factory when the model appears in tests or seeders frequently
- On the model, add **`#[UseFactory({Model}Factory::class)]`** once the factory exists (and import the factory class)
- Keep factories realistic and aligned with domain defaults
- Avoid factories that hide important invariants; make critical fields explicit

---

## Casts: always explicit

Casts must always be defined when a field requires it.

Casts are declared in the **`casts()` method** — never the `$casts` property.

Examples:
- Enums:
    - `'status' => ProductStatus::class`
- Dates:
    - `'published_at' => 'datetime'`
- Prices:
    - `'price' => 'float'`
- Quantities:
    - `'quantity' => 'int'`
- JSON fields:
    - `'meta' => 'array'`

Rules:
- If a field benefits from casting, cast it.
- Prefer enum casts for constrained states instead of strings/ints.
- Every enum that backs a model field MUST be cast here — the enum contract is:
  typed enum property + `Rule::enum(...)` in the Data class (`docs/04` § Enums in
  Data classes), enum cast in the model's `casts()`, and `label()` for presentation
  (`docs/11` § Enum contract).
- Casts must be kept up to date as schema evolves.

---

## Clean models: no `@property` lines, no comments

Model files carry **no `@property`/`@property-read` lists, no class docblocks,
and no explanatory comments**. A model is attributes wiring, `$visible`,
`casts()`, and relation methods — nothing else to read.

The one docblock a model keeps: the PHPStan generics annotation on each
relation method (`@return BelongsTo<User, $this>`). It is required.

PHPStan/IDE metadata comes from **barryvdh/laravel-ide-helper** (dev dependency):

- `php artisan ide-helper:models --nowrite` generates `_ide_helper_models.php`
  with `@mixin` metadata for every model.
- Regenerate after every migration/schema change.
- Never use `--write` (it injects docblocks into model files) and never
  hand-write `@property` lists.
- Larastan resolves relation and cast types natively.

---

## Visible: always explicit

Models must always define `$visible` to control what can be serialized.

Rule:
- Every model defines `$visible` explicitly.

Benefits:
- prevents accidental leakage of internal attributes
- keeps API responses predictable
- forces intentional output design

Do not rely on implicit serialization behavior.

---

## What models should not contain

Models must not contain:
- workflows (create/cancel/approve flows)
- cross-entity coordination
- complex business rules that belong in domain services or actions
- output formatting for UI

Models may contain:
- relationship declarations
- casts
- accessors/mutators when they are persistence-level and not business workflows
- simple invariants local to the model when they are genuinely persistence-level

---

## Summary checklist

- Model extends shared base model (with `HasFactory` on the base model)
- Model uses traits for generic persistence logic (uuid, actor, soft deletes, etc.)
- Relationships are defined explicitly with correct relation types
- Custom `{Model}Collection` exists and model declares `#[CollectedBy(...)]`
- Observer exists (when needed) and model declares `#[ObservedBy(...)]` (not `observe()` in `boot()`)
- Policy declared with `#[UsePolicy(...)]`; factory with `#[UseFactory(...)]` when a factory exists
- Domain `{Model}Rule` exists when policies need object-state checks; policies reference the Rule class by name (not via the model)
- No `$fillable` is used anywhere
- Factory exists when needed for tests/seeders
- Casts are explicit for every field that needs them (enums, dates, floats, ints, arrays)
- `$visible` is explicitly defined
- Routes that bind a domain model use `{param:uuid}` (no `getRouteKeyName()` on the model for URL binding)
