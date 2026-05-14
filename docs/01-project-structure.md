# Project Structure

This document defines the canonical project structure used at Labrodev.

The structure is domain-centric, explicit, and designed for long-lived systems, modular growth, and AI-assisted development. It separates dashboard/UI concerns from core business logic and allows the core to live either inside the repository or as a standalone Composer package.

Structure is treated as a design and communication tool, not as an implementation detail.

---

## Core principle

Code location communicates responsibility.

A developer should understand what a piece of code does, and what it depends on, by looking at where it lives. If a developer needs to ask "where should this go?", the structure has already failed.

---

## High-level overview

The project is split into three major areas:

- Application delivery layers (`App/Layer/{Layer}` — e.g. `Api`, `Dashboard`; a repository may define additional layer roots under `App/Layer`)
- Core business logic (Core)
- Cross-cutting and shared abstractions (Core/Shared and Core/Support)

All delivery layers depend on Core.  
Core must not depend on any delivery layer.

---

## Logical vs physical structure

All references in this documentation to paths such as
`Core/Domain/<Domain>` or `App/Layer/<Layer>/<Domain>` describe
**logical architectural modules**, not fixed filesystem paths.

The physical location of these modules may vary depending on
project layout, packaging, or deployment strategy.

Valid physical mappings include, but are not limited to:

- `src/Core/Domain/<Domain>`
- `packages/<Package>/src/Core/Domain/<Domain>`
- `modules/Core/Domain/<Domain>`

In **this project**, the Core lives behind the `/src` folder of the
separate `core` package (checked out next to the application repo),
for example:

- `../core/src/Domain/<Domain>`
- `../core/src/Shared`
- `../core/src/Support`

All architectural rules, boundaries, and conventions apply
to the logical module regardless of its physical location.

Documentation and stubs must refer to modules by their
logical names, not by concrete filesystem paths.

IMPORTANT: Check Core in composer also. In many cases it's a repository package which is located on the same level as project folder (../core). In this codebase, that Core package exposes its code under a `/src` directory.

---

## Domain README

Each domain module must include a `README.md` file located at the root
of the domain module directory.

This applies regardless of the physical filesystem layout.

This README must describe:
- the domain purpose and context
- ubiquitous language (key terms)
- main models and responsibilities
- key workflows (names only)
- invariants and constraints

It must not contain implementation instructions or business logic code.

## App/Layer

The `App/Layer` namespace defines **delivery layers** of the application.
Each layer hosts application-facing logic related to HTTP/controllers, queries for listing data, exports, and view-specific models, scoped to a particular surface of the product.

### What a Layer is

A **Layer** is a distinct application interface or delivery surface.

Examples:
- `Api` is the machine-facing interface (JSON, webhooks, integrations)
- `Dashboard` is the primary human-facing web UI layer (Inertia, session-backed flows)

A repository may introduce additional layer names under `App/Layer/{Layer}`; they follow the same internal layout described below.

A Layer is **not** a second business domain and **not** an alternative home for rules. It is the place where the system adapts one interface to the Core:
- routes
- controllers
- read-side queries for that interface
- view/API shaping concerns
- interface-specific authorization wiring

The purpose of Layers is to let the same Domain be delivered through multiple interfaces without pushing UI/API concerns into Core and without duplicating business workflows.

### Why we use Layers

We use `App/Layer/{Layer}` because different interfaces usually need different:
- entry points
- response shapes
- screens or resources
- read models
- navigation flows
- authorization context

But those differences should not force us to split or duplicate Domain logic.

The Layer boundary keeps this explicit:
- interface concerns live in `App/Layer/*`
- business logic lives in `Core/*`
- the same Domain can be exposed through more than one Layer

This makes it possible, for example, for `App/Layer/Dashboard/Booking/*` and `App/Layer/Api/Booking/*` to expose different screens or payloads while still delegating mutations and business rules to the same `Core/Domain/Booking` module.

### Layer mirrors Domain, but does not replace it

`App/Layer/{Layer}/{Domain}` intentionally mirrors domain names because each interface usually needs a delivery-side home for the same business area.

That mirroring means:
- `App/Layer/Dashboard/Booking/*` is the Dashboard delivery code for the Booking domain
- `App/Layer/Api/Booking/*` is the Api delivery code for the same Booking domain
- both point to the same Core Booking logic

This is a **reflection of the Domain in the interface**, not a copy of the Domain itself.

Rules:
- Domain language stays aligned across Layers and Core
- delivery classes may differ per Layer
- business rules, workflows, invariants, and mutations do not move into the Layer
- if two Layers need the same business behavior, that behavior belongs in Core, not duplicated in each Layer

The structure is therefore:
- Core owns the business meaning of the Domain
- Layer owns how that Domain is exposed to a specific audience/interface

### Typical layers

| Layer root (`App/Layer/…`) | Role |
|---------------------------|------|
| `Api` | Programmatic surface (JSON resources, webhooks, machine clients) |
| `Dashboard` | Primary browser UI (Inertia pages, forms, authenticated flows) |

Additional layer roots are allowed when the product needs a separate delivery surface; they reuse the same folder layout under `App/Layer/<Layer>/{Domain}/`.

Each layer follows the same internal structure:

- `App/Layer/<Layer>/{Domain}/`
    - `Controllers`
    - `JsonControllers`
    - `Exports`
    - `IndexQueries`
    - `ViewModels`

Delivery layers are responsible for:
- receiving user or client input
- preparing data for presentation
- orchestrating read-side queries
- delegating business operations to Core Actions or Orchestrators

They must not contain business rules.

### Organizing a Layer module

Each `App/Layer/{Layer}/{Domain}` module is organized by delivery responsibility, not by business behavior:

- `Controllers` for page/HTTP endpoints
- `JsonControllers` for in-page JSON interactions
- `IndexQueries` for read-side listing/filter/table queries
- `ViewModels` for presentation-ready output
- `Exports` for CSV/Excel/PDF output

This organization exists because a Layer solves interface problems:
- how to receive input
- how to authorize access at the boundary
- how to fetch and shape read-side data for that interface
- how to return the correct response type

It must not become a shadow domain structure with its own Actions, Services, Rules, or Policies that duplicate Core.

### Layer module layout (quick reference)

| Folder under `App/Layer/<Layer>/{Domain}/` | Responsibility |
|-------------------------------------------|----------------|
| `Controllers/` | Invokable HTTP endpoints; thin; class-level `#[Authorize]` when the subject is known up front |
| `JsonControllers/` | Invokable JSON for in-page interactions; same auth pattern; routes often under a `json/` prefix |
| `IndexQueries/` | Read-side listings, filters, pagination; never mutate domain state |
| `ViewModels/` | Presentation-ready props; no business rules |
| `Exports/` | CSV/Excel/PDF; may use IndexQueries/ViewModels; never mutate domain state |

---

### Controllers

Controllers in `App/Layer/<Layer>/{Domain}/Controllers` are **single-responsibility, invokable** classes. Each route corresponds to a dedicated controller with a single `__invoke()` method. Controllers handle input and output only: they delegate business behavior to Core Actions or Orchestrators and return responses or views. They should remain thin and boring.

**Pattern:** One HTTP endpoint maps to one invokable controller class. CRUD flows use a family of controllers per resource (e.g. `BookingIndexController`, `BookingCreateController`, `BookingStoreController`, `BookingShowController`, `BookingEditController`, `BookingUpdateController`, `BookingDeleteController`). Single-page or settings flows use one controller per page or command (e.g. `CompanyInformationIndexController`, `CompanyInformationUpdateController`, `EmployeeSelectionEditController`, `EmployeeSelectionUpdateController`).

**Naming:** Use `{Resource}{Action}Controller` where Action is the intent: `Index`, `Show`, `Create`, `Store`, `Edit`, `Update`, `Delete` for CRUD; or `Edit`/`Update` for form pages. Controllers extend `Controller` or `SupplierController` (when `resolveSupplier()` is required). The exact class shape is defined by the Layer controller stubs in `stubs/app/dashboard/controllers/*.stub`.

**Authorization (invokable controllers):** Use Laravel’s class-level **`#[Authorize(...)]`** attribute (`Illuminate\Routing\Attributes\Controllers\Authorize`) instead of calling `$this->authorize(...)` inside `__invoke()`, so the gate runs as controller middleware and matches the single-action route. Arguments follow `Gate::authorize` (policy ability + model class for “list/create”, or ability + route parameter name for route-model-bound instances, e.g. `#[Authorize('update', 'booking')]`).

**Dashboard layer conventions:** policy ability names, permission prefixes, and tenant scoping are project-specific. Document them in the domain README or layer README; wire them consistently with `#[Authorize(...)]` and Spatie permissions (and domain `*Rule` gates such as `*Rule::isEditable` where applicable).

**Exceptions — keep `$this->authorize(...)` in the method body when:**
- The subject is only known **after** runtime setup (e.g. **Configuration** controllers: fetch `ConfigurationSet` via `ConfigurationSetFetcher`, then authorize against that instance).
- The subject comes from **`resolveSupplier()`** (or similar) before any route-model binding (e.g. company information, external settings: authorize after resolving the `Supplier`).

**Example (invokable controller):**

```php
use Illuminate\Routing\Attributes\Controllers\Authorize;

#[Authorize(BookingPolicy::PERMISSION_VIEW, Booking::class)]
final class BookingIndexController extends Controller
{
    public function __invoke(BookingIndexQuery $query): Response
    {
        $bookings = $query->paginate(request()->get('per_page', 50))->withQueryString();
        $viewModel = new BookingIndexViewModel(items: $bookings);

        return Inertia::render('bookings/index', $viewModel->toArray());
    }
}
```

Route registration uses the invokable class directly: `Route::get('bookings', BookingIndexController::class)->name('bookings.index');`

### JsonControllers

JsonControllers in `App/Layer/<Layer>/{Domain}/JsonControllers` serve JSON responses for in-page interactions that do not require a full Inertia page visit (e.g. wizard steps, autocomplete, dynamic calculations).

They follow the same rules as regular controllers: invokable, single-responsibility, thin, delegate to Core, and **authorize with class-level `#[Authorize(...)]`** like other invokable controllers. The difference is that they return `JsonResource` or `AnonymousResourceCollection` instead of `Inertia::render(...)`.

JsonController routes are nested under a `json/` prefix within the resource route group and run under the same `web` middleware group (session + CSRF protection).

**Naming:** `{Entity}{Verb}Controller` where verb describes the operation: `Get`, `Find`, `Calculate`, `Search`.

See `docs/11-inertia-react.md` section 15 for the full frontend integration pattern including CSRF handling.

### Exports

Exports contain logic for producing external representations (CSV, Excel, PDF, etc.) based on prepared data. They may depend on IndexQueries or ViewModels.

Exports must not mutate domain state.

### IndexQueries

IndexQueries in `App/Layer/<Layer>/{Domain}/IndexQueries` are read-side query objects used for listings, tables, filters, pagination, and dashboards.

They are optimized for presentation and may join multiple data sources. They must not mutate state.

### ViewModels

ViewModels prepare structured, presentation-ready data for views or APIs. They hide internal data structures and prevent leakage of domain internals into the UI.

ViewModels must not implement business rules.

---

## Core/Domain

The Core/Domain layer contains the heart of the system: business logic, rules, and domain language.

Each {Domain} represents a bounded context or business area.

Core/Domain code must not depend on `App/Layer`, controllers, or view logic.

Structure (alphabetical):

- Core/Domain/{Domain}/
    - Actions
    - Casts
    - Collections
    - Data
    - Enums
    - Events
    - Exceptions
    - Factories
    - Jobs
    - Models
    - Observers
    - Orchestrators
    - Payloads
    - Pipelines
    - Policies
    - Queries
    - Resources
    - Rules
    - Services
    - Traits
    - Utilities

Below are responsibilities for each directory.

---

### Actions

Actions represent explicit business use cases. They describe what the system does.

Actions coordinate domain logic and infrastructure but do not contain presentation concerns.

Actions are the primary entry point for business behavior.

Naming rule reminder: actions start with the entity name and expose a single public method `__invoke()`.

**Where Actions live:** Actions exist **only in Core** (`Core/Domain/{Domain}/Actions/`). `App/Layer` must **not** define its own Action classes. Controllers receive user input, map it into Data objects, and **delegate mutations to Core Actions** by injecting them as method arguments and invoking them: `$productCreate(productCreateData: $productCreateData);`. If you need a new use case (e.g. “update travel data”), add the Action in Core and have the Layer controller call it—do not create `App/Layer/…/Actions/`.

---

### Casts

Casts define how values are transformed between raw representations and domain-friendly representations.

Casts may be used by:
- Eloquent models (persistence casting)
- Spatie Data objects (input mapping)
- domain utilities (consistent normalization/parsing)

Casts must not contain business decisions.

---

### Collections

Collections represent typed collections of domain objects or values and may encapsulate collection-specific behavior.

Collections exist to:
- make return types explicit
- keep collection logic out of Actions and Services
- improve readability and safety

Collections must not mutate domain state implicitly.

---

### Data

Data objects define explicit input and output contracts.

There are no Request classes in this system. Store and update validation rules live in Data classes (Spatie Data).

Data objects:
- define allowed input fields
- validate store/update input
- normalize and cast values into typed attributes
- provide a stable contract for Actions and Orchestrators

Data objects must not implement business rules.

---

### Enums

Enums express constrained sets of domain values and should be preferred over magic strings or integers.

Enums must not include "Enum" in the name (for example, ProductStatus, not ProductStatusEnum).

---

### Events

Events represent significant facts that happened in the domain. They are descriptive, not imperative.

Events must:
- carry data only
- contain no behavior
- avoid encoding workflows

Events should increase clarity, not hide execution flow.

---

### Exceptions

Exceptions represent domain-level error conditions and express failure in business terms.

Exceptions must not be used to represent invalid input shape; invalid input is handled at the Data validation boundary.

---

### Factories

actories define how model instances are created for tests, seeds, and development scenarios.

Factories are part of the persistence layer and exist to:
- provide consistent test data
- make tests expressive and readable
- encode realistic defaults for models
- avoid ad-hoc model creation logic in tests

Factories do not represent business behavior.

Factories may:
- define sensible default attributes
- generate valid, realistic data
- expose states for common variations (for example: inactive, archived, published)

Factories must not:
- implement business workflows
- call Actions or Orchestrators
- bypass invariants that exist in the domain
- encode business rules that belong in Core logic

Factories are a testing convenience, not a business abstraction.

### Jobs

Jobs represent asynchronous execution of domain behavior.

A Job is a thin infrastructure wrapper that delegates to Actions or Orchestrators. Jobs must not contain business logic.

Jobs exist to handle:
- background processing
- retries and backoff
- queue isolation
- time-shifted execution

If a Job contains logic that matters, that logic belongs in Actions, Orchestrators, Rules, or Services.

---

### Models

Models represent persistence and database records.

They may define relationships, casts, and simple invariants, but they are not the domain itself.

Models must not coordinate workflows or contain complex business rules.

---

### Observers

Observers may enforce invariants or persistence synchronization rules.

Observers must not implement workflows or business processes. They should be used sparingly and deliberately, because they introduce hidden behavior.

If behavior matters, it should be explicit and callable via an Action or Orchestrator.

---

### Orchestrators

Orchestrators coordinate multiple Actions or Services into higher-level workflows.

They are explicit workflow objects used to keep complex flows visible and testable.

Orchestrators are allowed to call Actions, but Actions should not call Orchestrators unless there is a very clear and consistent policy for it.

---

### Payloads

Payloads represent explicit, structured messages passed between steps of a workflow or pipeline.

Payloads exist to:
- replace anonymous arrays
- stabilize interfaces between stages
- make data movement explicit

Payloads must remain data-only and must not contain behavior.

---

### Pipelines

Pipelines model sequential processing steps and are used when behavior is naturally staged or composable.

Pipelines must:
- keep steps small and explicit
- use Payloads for stage communication
- avoid hidden mutations that make execution hard to trace

---

### Policies

Policies define authorization rules in business terms and express who is allowed to perform certain actions.

Policies must remain declarative and must not perform mutations.

---

### Queries

Queries encapsulate read-side logic related to the domain.

They must not mutate state and must not contain presentation-specific assumptions (tables, UI filters, export formats). Layer `IndexQueries` live only under `App/Layer/*`; Core `Queries` stay in `Core/Domain/*`.

**Return types (normative):**

- Domain Query methods must return `Illuminate\Database\Eloquent\Builder` by default. Do **not** return `int`, `Collection`, arrays, or other resolved results from methods that behave like composable query builders (e.g. `all()`, `active()`). A **narrow exception** exists for explicitly named terminal methods (e.g. `bookingExistsForEmployee(): bool`); see `docs/07-anti-patterns.md` §26.

For datetime handling across the codebase, see `docs/07-anti-patterns.md` §25 (`Illuminate\Support\Carbon`, not `CarbonImmutable`).

---

### Resources

Resources define how domain data is exposed to the outside world.

Resources are responsible for:
- shaping output
- protecting internal structures
- providing stable API contracts

Resources must not contain business logic.

---

### Rules

Rules represent small, reusable decision logic that is easy to test and reason about.

Rules exist to:
- keep business decisions explicit
- avoid duplicating logic across Actions and Services
- isolate complex conditions into named concepts

Rules must not perform mutations.

---

### Services

Services contain reusable domain logic that does not naturally fit inside a single Action.

Services should be stateless and explicit. Their names must be self-explanatory and verb-related (for example, EmailSender, PriceCalculator).

Services must not contain presentation concerns.
pdj
---

### Traits

Traits may be used for shared behavior inside a domain but should be used cautiously.

Prefer explicit composition or small shared classes when possible. Traits must not introduce hidden side effects.

---

### Utilities

Utilities contain low-level helper functions specific to the domain.

Utilities must not introduce hidden side effects and must not become dumping grounds for "misc" logic.

If a utility becomes widely reused across domains and has no domain terminology, consider moving it to Core/Support.

---

## Core/Support

Structure:

- Core/Support/
    - Helpers
    - Utilities

This layer contains technical helpers and infrastructure-oriented utilities shared across domains.

Core/Support must not contain business logic or domain rules.

---

## Core/Feature

Structure:

- Core/Feature/{FeatureName}/

Features contain isolated, cross-domain or highly specific logic that does not belong to any single domain.

Features:
- are self-contained
- may include any classes they require
- must not become dumping grounds

If a feature grows into a stable business concept, it should be promoted to a proper Domain.

---

## Core/Shared

Structure:

- Core/Shared/
    - Actions
    - Concerns
    - Models
    - Traits
    - etc

Core/Shared contains base abstractions used across multiple domains.

Shared code must remain generic and must not encode domain-specific rules.

Rule of thumb: if a class name includes a domain term (Product, Order, Invoice), it does not belong in Shared.

---

## Core/Infrastructure

### Structure of Infrastructure

- `Core/Infrastructure/{IntegrationName}/`

Infrastructure contains external integrations and technical adapters that connect the system to the outside world.

Examples include:

- Payment gateways
- Email providers
- SMS providers
- ERP integrations
- Marketplace APIs
- Webhook handlers
- File storage adapters
- Third-party REST or GraphQL clients

Infrastructure is **technical**, not business.

---

### Principles of Infrastructure

Infrastructure:

- Implements external communication
- Wraps third-party SDKs or HTTP clients
- Handles authentication, request formatting, retries
- Maps external data into internal domain-friendly structures
- May handle transport-level validation or normalization

Infrastructure must **not**:

- Contain business rules
- Decide business state transitions
- Implement workflows
- Replace Actions or Orchestrators
- Access `App/Layer`

Infrastructure is an adapter layer between the Domain and the external world.

---

### Dependency Direction of Infrastructure

- Domains may depend on Infrastructure via contracts (interfaces).
- Infrastructure may depend on external libraries and SDKs.
- Infrastructure must never depend on `App/Layer`.
- Infrastructure must not depend on specific Domain business rules.

If Infrastructure starts containing business decisions, that logic belongs in a **Domain Service** or **Orchestrator**.

---

## Dependency rules

The dependency direction is strict:

- `App/Layer` depends on Core
- Core/Domain may depend on Core/Shared and Core/Support
- Core must not depend on `App/Layer`

Violations of dependency direction are architectural errors.

---

## Closing note

This structure is intentionally explicit and verbose.

It favors clarity over minimalism, explicit flows over magic, and long-term maintainability over short-term convenience. When the structure no longer serves the system, it should be evolved deliberately rather than bypassed silently.
