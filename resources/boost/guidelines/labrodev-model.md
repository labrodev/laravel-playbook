# Labrodev Models & Migrations

Always-on law. A model is a **persistence object only** — it contains **nothing but relations, casts, and Attributes**: never logic of any kind (business, querying, rendering), never the domain itself, never a workflow coordinator. Every model ships with its full trio — **Policy, Observer, and Collection** — always. Anatomy and templates → the `labrodev-model` skill; per-file checks → `rules/models.md` and `rules/migrations.md`.

## Musts

- Every domain model lives in `Core/Domain/{Domain}/Models/{Model}.php` and **extends `Core/Shared/Models/BaseModel`**.
- **A model's entire content is**: the wiring class attributes, `$visible`, the `casts()` method, relation methods, and (when genuinely persistence-level) Eloquent `Attribute` accessors. Nothing else — no business logic, no querying logic, no rendering/formatting logic, no helper methods.
- **Every model has the mandatory trio**: a `{Model}Policy` (→ labrodev-authorization), a `{Model}Observer`, and a `{Model}Collection` — created with the model, not deferred.
- `BaseModel` sets `$guarded = ['*']` — the mechanism behind the architecture-wide mass-assignment ban (→ labrodev-core). Attributes are assigned explicitly, row by row, in Actions.
- Declare the table via the **`#[Table('actual_table')]` class attribute** when it differs from Laravel's snake_plural default — never `protected $table`.
- Observer, collection, policy, and factory are wired via **class attributes**, in order: `#[ObservedBy]`, `#[CollectedBy]`, `#[UsePolicy]`, `#[UseFactory]`; omit `#[UseFactory]` (and its import) until the factory exists.
- **No `@property` lines and no comments in models.** Column metadata for PHPStan/IDE comes from barryvdh/laravel-ide-helper **mixin mode**, regenerated after every schema change; each model carries exactly one `/** @mixin IdeHelper{Model} */` line — nothing else.
- Casts live in the **`casts()` method — never the `$casts` property** — covering every enum, `datetime`, `float`, `int`, and `array` (JSON) field; every enum backing a model field MUST be cast here.
- Every relation method declares the native Relation return type AND the PHPStan generics docblock (`@return BelongsTo<User, $this>`) — the ONE docblock kind allowed in a model.
- **User relations in Core point at the Core mirror model** `Core/Domain/User/Models/User` — never `App\Models\User` (→ labrodev-core dependency law).
- Every model defines **`$visible` explicitly** to control serialization; `BaseModel` hides audit columns via `$hidden` — concrete models never override `$hidden`.
- The `{Model}Collection` lives in `Core/Domain/{Domain}/Collections/`, declared with `#[CollectedBy(...)]`.
- The `{Model}Observer` lives in `Core/Domain/{Domain}/Observers/{Model}Observer.php`, is `final readonly`, wired only via `#[ObservedBy(...)]`, and handles persistence-adjacent side effects only (lifecycle logging by default).
- Model traits contain persistence-level logic only (actor/audit metadata, soft deletes); domain-specific traits live in the domain and are named accordingly.

## Migration musts

- Clear, singular model names; consistent snake_plural table names (`Booking` → `bookings`).
- Add a `uuid` column (unique, indexed) on any table whose model appears in URLs or needs a stable external identifier — the value is assigned in the create Action, not by the database or model.
- Keep schema predictable — no implicit behavior, no magic columns; every new column is mirrored in `casts()` when applicable and the ide-helper mixin regenerated.
- Include `timestamps()`; add `softDeletes()` when the model uses `SoftDeletes`.

## Must-nots

- Mass assignment stays banned architecture-wide, no model-level exceptions (→ labrodev-core).
- Never write `@property`/`@property-read` lists or explanatory comments in a model; `ide-helper:models --write` (full docblock injection) is equally forbidden — column metadata lives only in the generated mixin. Allowed annotations: the single `@mixin IdeHelper{Model}` line and relation `@return` generics.
- No logic of any kind in a model: no business workflows, queries, scopes, cross-entity coordination, calculations, or UI/rendering formatting (workflows → labrodev-action; reads → labrodev-query; presentation → labrodev-viewmodel-resource).
- Never generate UUIDs in the model, `boot()`, or a trait — UUIDs are assigned explicitly in the create Action (→ labrodev-action).
- Never override `getRouteKeyName()` — keep the default route key; routes bind by uuid with `{booking:uuid}` (→ labrodev-controller).
- Never register observers from `boot()` via `Model::observe()` — recursive boot; `#[ObservedBy]` is the only wiring.
- Never override `newCollection()` — `#[CollectedBy]` replaces it (Laravel 13+).
- Never skip a member of the trio — a model without its Policy, Observer, or Collection is incomplete.
- Object-state gates live in `{Model}Rule`, never on the model (→ labrodev-action).
- Observers must not change domain state, call mutating Actions/Services, or contain logic the main business flow depends on; Notifications go through the Event → Listener → Job → Notification chain (→ labrodev-action), never sent from an Observer.

Naming rules (→ labrodev-naming); file header contract (`declare(strict_types=1)`, `final`) and dependency direction (→ labrodev-core).
