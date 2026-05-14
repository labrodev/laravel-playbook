# Review Checklist

Use this checklist to review any PR or code change in a Labrodev Laravel project.  
Fail the review if any “Must” item is violated.

If a rule is unclear, consult `docs/index.md` and the relevant chapter.

---

## 1) Architecture and dependencies (Must)

- Must: `App/Layer/*` depends on `Core/*` only (never the other way)
- Must: `Core/*` never imports `App/Layer/*`
- Must: `Core/Shared` does not depend on any specific Domain
- Must: `Core/Support` contains only technical utilities and does not reference domain language
- Must: Cross-cutting workflows that do not belong in a single domain live in `Core/Feature/{FeatureName}/`, not duplicated across domains
- Must: No new `bind` / `singleton` / interface→implementation registrations in `AppServiceProvider` or other application service providers **unless** the task or implementation plan explicitly requires them (`docs/07-anti-patterns.md` §27)
- Must: No new interfaces, contracts, or abstraction layers **unless** the task or plan names them (`docs/07-anti-patterns.md` §27)

---

## 2) Delivery layer (`App/Layer`) (Must)

- Must: Controllers are thin (I/O + delegation only)
- Must: Layer controllers are single-responsibility, invokable classes
- Must: Invokable controllers use class-level `#[Authorize(...)]` unless the subject is only known after runtime setup
- Must: No business rules or workflows inside controllers
- Must: IndexQueries are read-only (no inserts/updates/deletes)
- Must: ViewModels are presentation-only (no domain decisions)
- Must: Exports are read-only (no domain mutation)

---

## 3) Controller -> Action pattern (Must)

- Must: Actions are injected as controller method arguments
- Must: No `app()`, `resolve()`, or `new` to obtain Actions
- Must: Controller invokes the Action as callable (e.g. `$productCreate(productCreateData: $productCreateData);`), not `$action->execute(...)`
- Must: No raw arrays passed into Core Actions/Orchestrators
- Must: No Request classes introduced anywhere
- Must: Read-side controllers do not introduce Request classes or Data classes for query input

---

## 4) Naming rules (Must)

- Must: Everything is singular (domains, folders, classes)
- Must: Actions are entity-first (ProductCreate, not CreateProduct)
- Must: Actions expose a single public `__invoke(...)` entry point (not `execute()` / `handle()`)
- Must: Jobs are verb-first and end with `Job` (SendEmailToCustomerJob)
- Must: Enums have no Enum suffix/prefix (ProductStatus)
- Must: Most domain component classes are entity-prefixed (ProductRule, ProductObserver, etc.)
- Must: Methods are verbal and explicit (fetchProduct, updateProductQuantity)
- Must: Model and Data attributes use snake_case identifiers ($product_id, $product_uuid)
- Must: Do not widen a required domain type to nullable to hide missing loads (e.g. keep `Product $product`, not `?Product $product`, when the contract requires a product) — fix the caller instead (`docs/07-anti-patterns.md` §28)

---

## 5) Inertia / React UI copy (Must)

- Must: User-visible strings use `t('Readable English')`, backed by Laravel `lang/*.json` and the frontend i18n bundle — see `docs/11-inertia-react.md` §10 and `docs/07-anti-patterns.md` §19
- Must: ViewModels do not pass `translations` (or equivalent) for Inertia page copy; pages resolve copy only via `t()` and `lang/` per `docs/11-inertia-react.md` §10

---

## 6) Data and validation (Must)

- Must: Validation exists only for store/update
- Must: Validation rules live in Data classes (Spatie Data)
- Must: Controllers do not validate
- Must: Casting uses Spatie Data casts and/or custom Cast classes
- Must: Validation targets raw input keys (e.g. product_uuid); casting happens after validation

---

## 7) Core/Domain structure (Must)

- Must: New domain code goes into the correct folder inside Core/Domain/{Domain}
- Must: No “random” folders created inside Core/Domain modules
- Must: Responsibilities match folder names (e.g. Rules contain rules, Services contain services)

---

## 8) Models and database rules (Must)

- Must: Models extend the shared base model
- Must: Models use traits for generic persistence concerns (UUID, actor, soft deletes, etc.)
- Must: Relationships are defined explicitly with correct relation type
- Must: No `$fillable` anywhere
- Must: No mass assignment (`fill()`, `create()` with arrays) for domain writes
- Must: Attributes are assigned explicitly (row by row) in Actions/Orchestrators
- Must: Casts are explicit for fields that need them (enums, dates, floats, ints, arrays)
- Must: `$visible` is defined explicitly

Collections:
- Must: Every model has a corresponding {Model}Collection in Collections
- Must: Model declares `#[CollectedBy({Model}Collection::class)]`

Policies / factories:
- Must: Model declares `#[UsePolicy(...)]` when a policy exists
- Should: `#[UseFactory(...)]` when a factory exists; base model uses `HasFactory`

Observers:
- Must: If an Observer exists, model declares `#[ObservedBy(...)]` (not `observe()` in `boot()`)
- Must: Observers do not implement workflows

Factories:
- Should: Factory exists when model is used in tests often
- Must: Factory does not encode workflows or call Actions/Orchestrators

---

## 9) Jobs, Observers, Events (Must)

Jobs:
- Must: Job contains no business logic
- Must: Job delegates to Action/Orchestrator
- Must: Retry/backoff/queue concerns live in Job only

Observers:
- Must: Observer does not coordinate workflows
- Must: Observer does not dispatch business-process jobs implicitly

Events:
- Must: Events are descriptive facts (data only)
- Should: Avoid event chains that hide primary flows

---

## 10) Datetime and domain Queries (Must)

- Must: Do not use `CarbonImmutable`; use `Illuminate\Support\Carbon` with explicit `copy()` when mutation safety matters (`docs/07-anti-patterns.md` §25)
- Must: `Core/Domain/.../Queries` methods return `Illuminate\Database\Eloquent\Builder` by default; do not return `int`, `Collection`, arrays, or other resolved results from builder-shaped query methods except the narrow, explicitly named terminal pattern in `docs/07-anti-patterns.md` §26

---

## 11) Testing (Should, but important)

- Should: Business behavior tested via Actions/Orchestrators (not via controllers)
- Should: Rules and Services tested as unit tests when non-trivial
- Should: Jobs tested as wrappers (delegation + configuration), not as business logic
- Should: HTTP tests verify wiring only, not business rules
- Should: Factories used for setup, not for behavior

---

## Review outcome guidance

- If any “Must” item fails: request changes.
- If multiple “Should” items fail: request changes unless justified.
- If a new pattern is introduced: it must be documented in `docs/` and aligned with `stubs/` (or the stub must be updated first).
