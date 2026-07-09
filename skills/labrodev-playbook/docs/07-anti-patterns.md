# Anti-Patterns

This document lists architectural and coding anti-patterns that are forbidden or strongly discouraged in Labrodev Laravel projects.

Anti-patterns are not style preferences. They are patterns that reliably create hidden coupling, unclear intent, fragile behavior, or long-term maintenance cost in our architecture.

If you see one of these in code review, treat it as a refactor requirement.

---

## 1) Business logic in the delivery layer

`App/Layer` is a delivery/UI layer. It must not own business decisions.

Forbidden examples:
- calculations that affect stored values inside controllers
- branching that decides business outcomes inside controllers
- mutating domain state inside IndexQueries or ViewModels
- writing domain state from Exports

Correct approach:
- map input into Data objects
- delegate mutations to Core Actions or Orchestrators
- keep the Layer focused on I/O, presentation, and delegation

---

## 2) Controllers that do work

Controllers must be thin.

Forbidden:
- complex workflows inside controllers
- coordinating multiple writes inside the controller
- hidden logic in private controller methods
- domain branching that changes business state

Correct approach:
- controller delegates to Action/Orchestrator
- controller returns a response/view
- controller contains no business rules

---

## 3) Hidden workflows in Observers

Observers are a common source of invisible side effects.

Forbidden:
- observers dispatching jobs that represent business processes
- observers calling multiple actions/services to “do things”
- observers coordinating workflows that are not visible in the call stack
- observers performing authorization-like decisions

Allowed (sparingly):
- persistence invariants
- low-level synchronization rules that are truly persistence-level

Correct approach:
- if behavior matters, it must be explicit via Actions or Orchestrators

---

## 4) Jobs that contain business logic

Jobs are infrastructure wrappers for async execution.

Forbidden:
- implementing business decisions inside a Job
- coordinating multi-model workflows inside a Job
- duplicating Action/Orchestrator logic inside Jobs

Correct approach:
- Job delegates to Action/Orchestrator
- Job owns retries/backoff/queue choice
- business logic stays in Core Actions/Orchestrators

---

## 5) Actions with ambiguous or hidden entry points

Actions must be invokable single-responsibility objects with one obvious entry point.

Forbidden:
- using `execute()` or `handle()` as the Action entry point
- multiple public methods on an Action (except constructor)
- calling Actions in a way that hides which method runs

Forbidden example:
- `$productCreate->execute($data);`

Correct approach:
- Actions expose a single public method: `__invoke(...)`
- call sites use the invokable explicitly: `$productCreate(productData: $data);` or `($productCreate)(...);`

Orchestrators continue to use `execute(...)` as their workflow entry point; this section applies to **Actions** (and single-operation Services — see `docs/02-naming.md`).

---

## 6) Resolving Actions manually instead of injecting them

Controllers must not manually resolve Actions.

Forbidden:
- `app(ProductCreate::class)`
- `resolve(ProductCreate::class)`
- `new ProductCreate()`

These approaches hide dependencies and make testing harder.

---

## 7) Incorrect Action usage in controllers

Controllers must not create or resolve Actions internally and must not pass raw arrays to Core.

Forbidden patterns:
- resolving Actions inside controller methods
- instantiating Actions manually
- passing raw input arrays into Actions/Orchestrators

---

## 8) Defining Actions under `App/Layer`

Actions belong **only in Core** (`Core/Domain/{Domain}/Actions/`).

Forbidden:
- creating `App/Layer/…/Actions/` or any Action class under the App layer
- implementing business mutations in `App/Layer` instead of Core

Correct approach:
- implement the Action in Core (e.g. `Core\Domain\Configuration\Actions\EmployeeSelectionUpdate`)
- have the Layer controller inject that Core Action and call it as invokable: `$employeeSelectionUpdate(...)`

The Layer handles I/O and delegation; Core owns all business behavior.

---

## 9) Correct Action usage in controllers (required pattern)

Actions must be injected explicitly into controller methods.

Required pattern:
- inject the **Core** Action as a controller method argument
- map input into a Data object (Core or Layer-local Data as appropriate)
- invoke the Action: `$productCreate(productData: $data);` (named arguments when more than one parameter)

Example pattern (conceptual):

- `public function store(ProductCreate $productCreate, ProductData $productData)`
- `$productCreate(productData: $productData);`

Controllers must remain thin and should not hide dependencies.

---

## 10) Mass assignment and fillable

Mass assignment is not used in this architecture.

Forbidden:
- `$model->fill($data)`
- `$model::create($attributes)`
- defining or relying on `$fillable`

Correct approach:
- assign attributes explicitly (row by row) in Actions/Orchestrators
- keep write intent visible and reviewable

---

## 11) “Shared” as a dumping ground

Core/Shared exists for truly generic abstractions.

Forbidden:
- placing domain-named classes in Shared (ProductSomething, OrderSomething)
- moving code into Shared just to avoid deciding where it belongs
- growing Shared into an unbounded “misc” module

Correct approach:
- if it contains domain language, it belongs in that domain
- if it is generic and reusable across domains, it can live in Shared

---

## 12) Support as a backdoor for business logic

Core/Support must remain technical.

Forbidden:
- business rules in Core/Support
- classes referencing domain nouns in Support
- using Support utilities to bypass domain boundaries

Correct approach:
- Support contains technical helpers only (formatting, serialization, IDs, etc.)
- business logic stays in Core/Domain

---

## 13) IndexQueries that mutate state

IndexQueries are read-side objects for Layer listings.

Forbidden:
- any insert/update/delete in IndexQueries
- calling Actions from IndexQueries
- doing domain mutation during listing generation

Correct approach:
- IndexQueries query and prepare data only
- mutations happen through Actions/Orchestrators

---

## 14) ViewModels that contain business rules

ViewModels exist to shape output for views or APIs.

Forbidden:
- business decisions inside ViewModels
- writing data from ViewModels
- complex branching that determines domain behavior

Correct approach:
- ViewModels format already-decided data for presentation
- decisions stay in Core

---

## 15) Event chains that hide critical flows

Events are descriptive facts, not workflow engines.

Forbidden:
- implementing critical business flow only through event listeners
- long listener chains that make flow impossible to trace
- relying on events to hide coupling between modules

Correct approach:
- primary workflows live in Actions/Orchestrators
- events can be used for secondary side effects that do not hide the main flow

---

## 16) Generic names that hide intent

Generic names reduce clarity and increase mistakes.

Forbidden class naming patterns:
- Manager
- Handler
- Processor
- Helper (except Core/Support/Helpers by design)
- Util

Correct approach:
- self-explanatory names (EmailSender, PriceCalculator, ProductEvaluationOrchestrator)
- prefer explicit domain prefixes for most component types

---

## 17) Readonly Data classes

Data classes (Spatie Laravel Data) must not be marked as `readonly`.

Forbidden:
- `final readonly class AreaData extends Data`
- `readonly final class AreaData extends Data`

Reason:
- Spatie Laravel Data's `Data` base class is not readonly
- PHP does not allow readonly classes to extend non-readonly classes

Correct approach:
- `final class AreaData extends Data`
- Data classes should be `final` but not `readonly`

---

## 18) Comments 

Comments are forbidden unless they provide **essential, non-obvious context**.

Do not add comments that:
- restate what the code already expresses
- explain trivial or self-evident logic
- describe *how* the code works instead of *why* it exists

Code must be written to be readable without commentary.

Comments are allowed only when they explain:
- architectural intent
- non-obvious constraints
- reasoning that cannot be expressed through code structure or naming

## 19) Incomplete UI copy pipeline (Laravel lang → i18n → `t()`)

User-facing copy for React/Inertia pages must follow **`docs/11-inertia-react.md` §10**: Laravel **`lang/*.json`** is the source of truth, the frontend i18n bundle (e.g. under `resources/js/i18n/`) is derived from or kept in sync with it, and components resolve strings only via **`t()`** — not ad hoc JSX literals as the primary source.

Forbidden:
- Missing or stale entries in **Laravel `lang/*.json`** (or whatever files feed the frontend dictionary) so `t()` cannot resolve a string the UI uses
- Referencing `t('…')` keys that do not exist in that pipeline
- Relying on hardcoded JSX strings for labels, titles, buttons, or empty states when `t()` is expected
- ViewModel `translations`, `getTranslations()`, or any equivalent map of UI strings passed as Inertia props for copy rendered in React — use **`docs/11-inertia-react.md` §10** (`lang/` → i18n → `t()`)

Correct approach:
- Add or update the string in **Laravel `lang/`** first, ensure the frontend dictionary includes it, then call **`t('…')`** using readable English source strings as keys where practical (not dotted slugs as the default style — see **`docs/11-inertia-react.md` §10**)

---

## 20) Exposing sensitive data through Inertia props

Inertia serializes all page props into a `data-page` JSON attribute in the HTML DOM. Everything passed to `Inertia::render()` is visible to anyone who inspects the page.

Forbidden:
- Passing raw Eloquent models or `$model->toArray()` to Inertia pages
- Including internal database IDs when UUIDs are the public identifier (unless `id` is explicitly displayed)
- Exposing password hashes, API tokens, secrets, credentials, or internal system flags
- Passing full eagerly-loaded relations that the page does not render
- Passing unfiltered collections without shaping them through Resources

Correct approach:
- Always pass models through Resources (explicit allowlist of fields)
- ViewModels decide *which* data; Resources decide *what fields*
- Review Resource output: every field must have a corresponding UI element
- Audit eager-loaded relations: nested Resources must also filter fields
- During development, inspect the `data-page` attribute in the browser to verify no sensitive data is exposed

See `docs/11-inertia-react.md` section 14 for the full frontend perspective.

---

## 21) Manual Data construction from Request in controllers

Data objects must be injected directly into controller method signatures. Spatie Laravel Data resolves them from the request automatically.

Forbidden:
- `$data = MyData::from($request->all())`
- `$data = MyData::from(['field' => $request->input('field'), ...])`
- Injecting `Illuminate\Http\Request` to manually construct a Data object

Correct approach:
- Type-hint the Data class as a controller parameter: `public function __invoke(MyData $myData, ...)`
- Laravel's service container + Spatie Data handle the mapping from request input to typed Data object

Manual construction is redundant, error-prone, and hides the input contract.

---

## 22) Frontend workarounds for missing backend functionality

When the backend does not yet provide data, logic, or behavior that a feature requires, the correct response is to implement it on the backend — not to fake it on the frontend.

Forbidden:
- Hardcoding data, options, calculations, or conditional logic in React to simulate backend behavior
- Filtering, sorting, or transforming data in the frontend when it should happen in an IndexQuery, ViewModel, or Resource
- Catching errors silently and rendering a fallback instead of surfacing the problem
- Adding frontend-only business rules that should live in Core (Actions, Services, Rules)

Correct approach:
- If the backend does not provide what the frontend needs, create the necessary backend code (ViewModel, Action, Service, Resource, JsonController)
- If the backend change is out of scope, explicitly flag the gap — do not silently work around it

The frontend is a rendering surface. It makes backend decisions visible. It does not replace them.

---

## 23) Internal database IDs in Data classes

Data classes must never expose internal database IDs (`_id` fields).

Forbidden:
- `public ?int $service_id = null`
- `public ?int $customer_id = null`
- Any `_id` suffixed property that references a related model

Correct approach:
- Use the entity name as the property with a UUID caster:
  `#[WithCast(ServiceUuidCaster::class)] public ?Service $service = null`
- The frontend sends the UUID, the caster resolves it to a model
- Internal IDs never leave the backend

Internal IDs are implementation details. UUIDs are the public contract.

---

## 24) Computed fields in Data classes

Data classes represent input contracts — what the client sends.

Forbidden:
- Including fields that are calculated by the backend (service_time, price, status)
- Including fields that the client never submits
- Expecting the frontend to compute and pass derived values

Correct approach:
- Only include fields the client actually provides
- Calculate derived values in Actions or Orchestrators
- Set default status values in Actions, not in Data classes

If the client doesn't send it, the Data class doesn't declare it.

---

## 25) Do not use `CarbonImmutable`

Do **not** use `CarbonImmutable`.

Forbidden:
- `CarbonImmutable` as a project-wide datetime type (or anywhere a normal `Carbon` instance is expected)

Correct approach:
- use `Illuminate\Support\Carbon`
- when mutation safety matters, use explicit `copy()` on the Carbon instance (do not rely on immutable types for “safety by default”)

---

## 26) Domain `Queries` must return `Builder` only (resolved results forbidden)

Domain Query classes are the single place for model query composition. **Methods must return `Illuminate\Database\Eloquent\Builder` by default** — do **not** return `int`, `Collection`, arrays, or other resolved results from domain query methods that behave like composable query entry points.

Forbidden:
- methods named like filters or scopes (`all()`, `forUser()`, `active()`) that return `int`, `Collection`, arrays, or other resolved results instead of a `Builder`

Narrow exception (allowed):
- explicitly named terminal methods whose names state the outcome (e.g. `bookingExistsForEmployee(): bool`) and whose body is a thin terminal on the builder (`exists()`, `count()`, `value()`, …)

Correct approach:
- prefer returning a `Builder` and let the caller run `->get()` / `->paginate()` / `->exists()` at the edge
- if you need a reusable existence or count check, add a dedicated method whose name documents the scalar return type

---

## 27) Interface bindings and service-provider registration without a plan

Do not introduce **container bindings** (interface → implementation, `singleton`, `bind`, contextual binding, etc.) in **`AppServiceProvider`** or other **application service providers** unless the **task or implementation plan** explicitly tells you to register that abstraction.

Do not introduce **interfaces, contracts, or “concern” types** as indirection layers **unless** the task or plan names the contract, the implementation, and where it is wired.

Forbidden:
- “Future-proofing” by binding interfaces to classes in a provider with no product requirement
- Auto-wiring abstractions because other codebases do it

Correct approach:
- Prefer **concrete constructor injection** of domain/layer classes already used in the codebase
- If a plan requires an abstraction, implement exactly what it specifies (contract + implementation + registration location)

---

## 28) Nullable-widening for required domain objects

If business logic **requires** a loaded domain object (e.g. `Product $product`), do **not** widen the type to `?Product` (or optional wrappers) to hide missing relations, lazy-loading mistakes, or incomplete ViewModels/Queries.

Forbidden:
- Changing `Product $product` to `?Product $product` when every valid call path must have a product
- Early-returning or no-op paths that mask a caller bug instead of fixing loading or the call contract

Correct approach:
- Ensure the **caller** loads the model/relation (query, ViewModel, Action) so non-nullability is honest
- If “absence” is a real business outcome, model it explicitly in the plan (separate method, explicit `Optional`-style type, or a different use case) — do not silently weaken a required parameter

---

## Summary

If you remember only a few rules:

- The delivery layer (`App/Layer`) is not business logic
- Controllers delegate, they do not decide
- Actions use `__invoke()` and are injected into controllers
- Jobs are wrappers, not brains
- Observers are not workflow engines
- No mass assignment and no `$fillable`
- Shared and Support must not become dumping grounds
- UI copy: Laravel `lang/*.json` → frontend i18n → `t()`; see `docs/11-inertia-react.md` §10 and `docs/07-anti-patterns.md` §19
- Never pass raw models to Inertia — always use Resources as the security boundary
- Backend gaps are backend tasks — the frontend does not fake functionality
- Data classes use UUID casters for relations — never internal IDs
- Data classes only contain fields the client actually sends
- Prefer `Illuminate\Support\Carbon` (never `CarbonImmutable`); use `copy()` when mutation safety matters
- Domain Query methods return `Builder` only unless the method is an explicitly named terminal read (§26)
- No interface/provider bindings or new contracts unless the task or plan says so (§27)
- Do not nullable-widen required domain parameters to hide missing data (§28)
