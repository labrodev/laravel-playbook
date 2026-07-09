---
name: labrodev-core
description: "Use when writing or reviewing ANY PHP code in a Labrodev Laravel project (Laravel 13+, PHP 8.5): creating a class of any kind, deciding which folder/namespace code belongs in, checking dependency direction between App/Layer and Core, applying final/readonly rules, or auditing code for architectural anti-patterns. This is the always-on foundation every other labrodev-* skill assumes."
license: MIT
metadata:
  author: labrodev
---

# Labrodev Core — structure, boundaries, headers, immutability

Part of the Labrodev playbook skill set — this is one of the two foundation skills (with labrodev-naming) that every other labrodev-* skill assumes. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

## Philosophy in six lines

- Readability beats cleverness: explicit code over magic, boring over smart, duplication over premature abstraction.
- Structure is a map of intent: code location communicates responsibility. If you must ask "where does this go?", the answer below is normative.
- Controllers are I/O, not business logic. Every piece of business logic has an intentional home in Core.
- Eloquent models are persistence, not the domain. Anemic models, explicit orchestration.
- Explicit flows over hidden side effects: if behavior matters, it must be visible in the call stack.
- Optimize for change velocity: shallow abstractions, safe refactoring, easy deletion.

## Musts

- Every PHP file starts with `<?php`, blank line, `declare(strict_types=1);`, blank line, `namespace ...;` matching the folder path.
- Every concrete class is `final`. Only abstract base classes explicitly designed for extension (e.g. an abstract `BaseModel` in `Core/Shared/Models`) and classes required non-final by the framework are exempt. PHP traits cannot be `final` — do not flag them.
- Dependency direction is strict: `App/Layer/*` depends on `Core/*`. Never the reverse.
- Mutations follow the write golden path; reads follow the read golden path (see below).
- Business logic lives only in Core (Actions, Services, Rules, Orchestrators, Policies).
- `Core/Shared` stays generic; `Core/Support` stays technical. A class whose name contains a domain noun (Booking, Invoice, Order) belongs in that domain, never in Shared or Support.
- Datetime type is `Illuminate\Support\Carbon`; use explicit `->copy()` when mutation safety matters.

## Must-nots

- Core code must never import an `App\Layer\*` namespace. `Core/Shared` must not depend on any specific Domain. `Core/Support` must not depend on Domain, Feature, or delivery layers. `Core/Infrastructure` must never access `App/Layer` and must not contain business rules.
- No business logic in `App/Layer`: no calculations that affect stored values, no business branching, no domain mutation in controllers, IndexQueries, ViewModels, or Exports.
- No `Actions/`, `Services/`, `Queries/`, `Rules/`, or `Policies/` folders under `App/Layer` — those class types exist only in Core.
- No mass assignment anywhere: no `fill()`, no `Model::create()`, no `$fillable` (mechanics → see the labrodev-model skill).
- No manual resolution of Actions in controllers: no `app(...)`, `resolve(...)`, or `new` — inject them as method arguments.
- No `CarbonImmutable`.
- No comments that restate code. Comments are allowed only for essential, non-obvious context (architectural intent, non-obvious constraints).
- No interface bindings, contracts, or service-provider registrations unless the task or plan explicitly names the contract, the implementation, and where it is wired. Prefer concrete constructor injection.
- No nullable-widening of required domain parameters (`Booking $booking` must not become `?Booking $booking` to hide a caller/loading bug). Fix the caller; model real absence explicitly.
- No frontend workarounds for missing backend behavior — implement the backend piece or flag the gap.
- No app-specific ownership/scoping vocabulary or logic in playbook code — such concerns (per-account data scoping, current-context services) are project-specific and live in the app, never in playbook templates or rules.

## Project structure (canonical)

Logical layout. Physically, Core often lives in a separate Composer package checked out next to the app (e.g. `../core/src/Domain/...`) — check `composer.json`. All rules apply to the logical module regardless of physical location.

```
src/Core/
├── Domain/{Domain}/            business logic, one bounded context per domain
│   ├── Actions/  Casts/  Collections/  Data/  Enums/  Events/  Exceptions/
│   ├── Factories/  Jobs/  Models/  Observers/  Orchestrators/  Payloads/
│   ├── Pipelines/  Policies/  Queries/  Resources/  Rules/  Services/
│   └── Traits/  Utilities/
├── Feature/{FeatureName}/      isolated cross-domain workflows; promote to a Domain when stable
├── Infrastructure/{Integration}/  external adapters (payment, email, ERP, APIs); technical, not business
├── Shared/                     generic base abstractions (base models, concerns, traits); no domain nouns
└── Support/                    technical helpers/utilities only; no business logic, no domain language

app/Layer/{Layer}/{Domain}/     delivery surfaces, e.g. Dashboard (Inertia UI), Api (JSON)
├── Controllers/                invokable HTTP endpoints, thin dispatchers
├── JsonControllers/            invokable JSON endpoints for in-page interactions
├── IndexQueries/               read-side listings/filters/pagination; never mutate
├── ViewModels/                 presentation-ready props; no business rules
└── Exports/                    CSV/Excel/PDF output; may use IndexQueries/ViewModels; never mutate
```

- A Layer is a delivery surface, not a second business domain. `App/Layer/Dashboard/Booking/*` and `App/Layer/Api/Booking/*` are two interfaces to the same `Core/Domain/Booking` module.
- Layers mirror domain names but never duplicate domain behavior. If two Layers need the same business logic, it belongs in Core.
- There is exactly one Request-less input boundary: controllers map raw input into Spatie Data objects (→ see the labrodev-data skill).

## Dependency rules

Allowed:

- `App/Layer/*` → `Core/*`
- `Core/Domain/*` → `Core/Shared`, `Core/Support`, and other `Core/Domain/*` modules when the relationship is explicit (types, models, queries)
- `Core/Feature/*` → `Core/Domain/*`, `Core/Shared`, `Core/Support`
- `Core/Shared` → `Core/Support`
- `Core/Domain/*` → `Core/Infrastructure` via explicit contracts only (and only when the plan calls for them)

Forbidden:

- `Core/*` → `App/Layer/*` (any direction into the delivery layer)
- `Core/Shared` → any specific Domain
- `Core/Support` → Domain, Feature, or delivery layers
- Domain code depending on HTTP concepts, the request lifecycle, ViewModels, Resources-for-views, or exports

Violations are architectural errors, not style issues.

## Golden paths

Write side (mutations):

```
Layer Controller → Data object → Core Action → Models
Layer Controller → Data object → Core Orchestrator → Actions → Models
```

Read side (queries):

```
Layer Controller → Layer IndexQuery → ViewModel → Resource → response
Core-internal reads → Core Domain Queries
```

IndexQueries are delivery-layer objects and must never be used from Core. Core Queries must never carry presentation assumptions (→ see the labrodev-query skill).

## File header contract (canonical templates)

Naming pattern: namespace mirrors the folder path; class name states intent. All naming rules → see the labrodev-naming skill.

Core class header — worked example, Booking domain (`Core/Domain/Booking/Actions/BookingCreate.php`):

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
        // full Action anatomy → see the labrodev-action skill
    }
}
```

Layer class header — worked example (`app/Layer/Dashboard/Booking/ViewModels/BookingIndexViewModel.php`):

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\ViewModels;

use Spatie\ViewModels\ViewModel;

final class BookingIndexViewModel extends ViewModel
{
    // full ViewModel anatomy → see the labrodev-viewmodel-resource skill
}
```

The root namespace prefix may differ when Core is a separate package (e.g. `Vendor\Core\Domain\Booking\...`) — the segment structure after the root is fixed.

## Immutability rules

- Modifier order is always `final readonly class`, never `readonly final class`.
- Apply `readonly` when a class has no mutable instance state — the default for constructor-injected stateless classes: Actions, Services, Orchestrators, Events.
- Do NOT apply `readonly` to:
  - Spatie Data subclasses — the base `Data` class is not readonly and PHP forbids a readonly class extending a non-readonly one. Correct: `final class BookingData extends Data`.
  - Any class extending a non-readonly base: Models, Controllers, ViewModels, Resources, IndexQueries built on framework bases.
  - Static-only classes (e.g. a Rule exposing only static methods) — `readonly` is meaningless without instance properties; write `final class BookingRule`.
- Mutability and inheritance are intentional decisions, never defaults. If a class is not `final` or not `readonly` where it could be, there must be a stated reason.

## Anti-patterns (enforceable list)

Treat any of these in review as a refactor requirement:

1. Business logic in `App/Layer` (controllers, IndexQueries, ViewModels, Exports deciding or mutating).
2. Controllers that do work: workflows, multi-write coordination, hidden logic in private methods (controller anatomy → see the labrodev-controller skill).
3. Hidden workflows in Observers: dispatching business jobs, chaining actions, authorization-like decisions. Observers may enforce persistence invariants only (→ see the labrodev-model skill).
4. Jobs containing business logic — Jobs are async wrappers that delegate to Actions/Orchestrators and own only retries/backoff/queue config.
5. Actions with `execute()`/`handle()` or multiple public entry points — Actions expose a single `__invoke()` (→ see the labrodev-action skill).
6. Manually resolving or instantiating Actions in controllers; passing raw arrays into Core.
7. Mass assignment in any form (`fill()`, `::create()`, `$fillable`).
8. `Core/Shared` or `Core/Support` as dumping grounds; domain nouns in either.
9. Generic class names (`Manager`, `Handler`, `Processor`, `Util`) → see the labrodev-naming skill.
10. Critical business flow implemented only through event listener chains — primary workflows live in Actions/Orchestrators; events are descriptive facts with no behavior.
11. Comments restating what code expresses.
12. `CarbonImmutable` anywhere.
13. Domain Query methods returning resolved results instead of `Builder` → see the labrodev-query skill.
14. Data-class violations (readonly Data, internal `_id` fields, computed fields, manual `::from($request)` construction) → see the labrodev-data skill.
15. Raw models or sensitive fields in Inertia props → see the labrodev-viewmodel-resource skill.
16. Unplanned interface bindings / provider registrations (§ Must-nots above).
17. Nullable-widening required domain parameters.

## Two-zone legacy policy

Every Labrodev repo has two zones:

- **Playbook zone** — `src/Core` (or the Core package) and `app/Layer`. All labrodev-* rules apply strictly. All new code goes here.
- **Legacy/vendor zone** — framework and starter-kit scaffolding under `app/Http`, `app/Models`, and `app/Actions/Fortify` (Fortify actions, starter middleware, default `User` wiring). This zone is **exempt** from playbook rules: do not flag it in review, and do not refactor it to playbook style unless the task explicitly asks.

Never add new business code to the legacy zone. When a legacy-zone concept needs real behavior, build it in Core and delegate.

## Edge cases

- **Logical vs physical paths**: `Core/...` references are logical modules. Physical homes vary (`src/Core/...`, `packages/*/src/...`, `../core/src/...`). Verify via `composer.json`; apply rules to the logical module either way.
- **Cross-domain needs**: one workflow spanning multiple domains with no natural home goes in `Core/Feature/{FeatureName}` with a small explicit API. When it stabilizes into a business concept, promote it to a proper Domain.
- **Abstract bases**: an abstract class in `Core/Shared` (e.g. `BaseModel`) is the sanctioned exception to `final`. Concrete subclasses are still `final`.
- **Utilities drift**: a domain Utility reused across domains with no domain terminology moves to `Core/Support`.
- **Infrastructure creep**: the moment an Infrastructure adapter makes a business decision, that logic moves to a Domain Service or Orchestrator.

## Review checklist

- [ ] Does every PHP file open with `declare(strict_types=1)` and a namespace matching its folder?
- [ ] Is every concrete class `final` (abstract Shared bases being the only exception)?
- [ ] Is `readonly` applied where state is immutable, in `final readonly` order, and absent from Data subclasses, framework-extending classes, and static-only classes?
- [ ] Does no `Core/*` file import an `App\Layer\*` namespace (and Shared/Support import no Domain)?
- [ ] Does each class live in the folder matching its responsibility — and are there no `Actions/`, `Services/`, `Queries/`, `Rules/`, or `Policies/` folders under `App/Layer`?
- [ ] Do mutations follow Controller → Data → Action → Model, and reads Controller → IndexQuery → ViewModel → Resource?
- [ ] Are `Core/Shared` and `Core/Support` free of domain nouns and business rules?
- [ ] Is the code free of the enforceable anti-patterns (mass assignment, manual Action resolution, `CarbonImmutable`, unplanned bindings, restating comments)?
- [ ] Is legacy starter code (`app/Http`, `app/Models`, `app/Actions/Fortify`) left untouched, with no new business code added there?
