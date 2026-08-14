# Labrodev Models & Migrations

Always-on law. A model is a **persistence object only**: state + casts + relations — never the domain itself, never a workflow coordinator. Every model has a Collection; Observers are optional and handle persistence-adjacent side effects only. Anatomy and templates → the `labrodev-model` skill; per-file checks → `rules/models.md` and `rules/migrations.md`.

## Musts

- Every domain model lives in `Core/Domain/{Domain}/Models/{Model}.php` and **extends `Core/Shared/Models/BaseModel`**.
- `BaseModel` sets `$guarded = ['*']` — the mechanism behind the architecture-wide mass-assignment ban (→ labrodev-core). Attributes are assigned explicitly, row by row, in Actions.
- Declare the table via the **`#[Table('actual_table')]` class attribute** when it differs from Laravel's snake_plural default — never `protected $table`.
- Observer, collection, policy, and factory are wired via **class attributes**, in order: `#[ObservedBy]`, `#[CollectedBy]`, `#[UsePolicy]`, `#[UseFactory]`; omit `#[UseFactory]` (and its import) until the factory exists.
- **No `@property` lines and no comments in models.** Column metadata for PHPStan/IDE comes from barryvdh/laravel-ide-helper **mixin mode**, regenerated after every schema change; each model carries exactly one `/** @mixin IdeHelper{Model} */` line — nothing else.
- Casts live in the **`casts()` method — never the `$casts` property** — covering every enum, `datetime`, `float`, `int`, and `array` (JSON) field; every enum backing a model field MUST be cast here.
- Every relation method declares the native Relation return type AND the PHPStan generics docblock (`@return BelongsTo<User, $this>`) — the ONE docblock kind allowed in a model.
- Every model defines **`$visible` explicitly** to control serialization; `BaseModel` hides audit columns via `$hidden` — concrete models never override `$hidden`.
- Every model has a `{Model}Collection` in `Core/Domain/{Domain}/Collections/`, declared with `#[CollectedBy(...)]`.
- Observers, when present, live in `Core/Domain/{Domain}/Observers/{Model}Observer.php`, are `final readonly`, wired only via `#[ObservedBy(...)]`.
- Model traits contain persistence-level logic only (actor/audit metadata, soft deletes); domain-specific traits live in the domain and are named accordingly.

## Migration musts

- Clear, singular model names; consistent snake_plural table names (`Booking` → `bookings`).
- Add a `uuid` column (unique, indexed) on any table whose model appears in URLs or needs a stable external identifier — the value is assigned in the create Action, not by the database or model.
- Keep schema predictable — no implicit behavior, no magic columns; every new column is mirrored in `casts()` when applicable and the ide-helper mixin regenerated.
- Include `timestamps()`; add `softDeletes()` when the model uses `SoftDeletes`.

## Must-nots

- Mass assignment stays banned architecture-wide, no model-level exceptions (→ labrodev-core).
- Never write `@property`/`@property-read` lists or explanatory comments in a model; `ide-helper:models --write` (full docblock injection) is equally forbidden — column metadata lives only in the generated mixin. Allowed annotations: the single `@mixin IdeHelper{Model}` line and relation `@return` generics.
- No business workflows, queries, scopes, cross-entity coordination, complex calculations, or UI formatting in a model (workflows → labrodev-action; reads → labrodev-query).
- Never generate UUIDs in the model, `boot()`, or a trait — UUIDs are assigned explicitly in the create Action (→ labrodev-action).
- Never override `getRouteKeyName()` — keep the default route key; routes bind by uuid with `{booking:uuid}` (→ labrodev-controller).
- Never register observers from `boot()` via `Model::observe()` — recursive boot; `#[ObservedBy]` is the only wiring.
- Never override `newCollection()` — `#[CollectedBy]` replaces it (Laravel 13+).
- Object-state gates live in `{Model}Rule`, never on the model (→ labrodev-action).
- Observers must not change domain state, call mutating Actions/Services, or contain logic the main business flow depends on.

Naming rules (→ labrodev-naming); file header contract (`declare(strict_types=1)`, `final`) and dependency direction (→ labrodev-core).
