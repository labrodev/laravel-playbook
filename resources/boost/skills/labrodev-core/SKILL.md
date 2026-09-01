---
name: labrodev-core
description: "Use when writing or reviewing ANY PHP code in a Labrodev Laravel project (Laravel 13+, PHP 8.5): creating a class of any kind, deciding which folder/namespace code belongs in, checking dependency direction between App/Layer and Core, applying final/readonly rules, or auditing code for architectural anti-patterns. This is the always-on foundation every other labrodev-* skill assumes."
license: MIT
metadata:
  author: labrodev
---

# Labrodev Core — structure, boundaries, headers, immutability

Part of the Labrodev playbook. **The law for this skill lives in the always-on `labrodev-core` guideline** (philosophy, structure, dependency law, golden paths, global code rules, anti-patterns, legacy zones) — this skill holds the craft: structure detail, canonical header templates, and edge cases. Per-file checks are folded into each component's rule file under `rules/`.

## Project structure (canonical)

Logical layout. Physically, Core often lives in a separate Composer package checked out next to the app (e.g. `../core/src/Domain/...`) — check `composer.json`. All rules apply to the logical module regardless of physical location.

```
src/Core/
├── Domain/{Domain}/            business logic, one bounded context per domain
│   ├── Actions/  Casts/  Collections/  Data/  Enums/  Events/  Exceptions/ (→ labrodev-exception skill)
│   ├── Factories/  Jobs/  Models/  Observers/  Payloads/
│   ├── Pipelines/  Policies/  Queries/  Rules/  Services/
│   └── Traits/  Utilities/
├── Feature/{FeatureName}/      isolated cross-domain workflows; promote to a Domain when stable
├── Infrastructure/{Integration}/  external adapters (payment, email, ERP, APIs); technical, not business → see the labrodev-infrastructure skill
├── Shared/                     generic base abstractions (base models, concerns, traits); no domain nouns
└── Support/                    technical helpers/utilities only; no business logic, no domain language

app/Layer/{Layer}/{Domain}/     delivery surfaces, e.g. Dashboard (Inertia UI), Api (JSON)
├── Controllers/                invokable HTTP endpoints, thin dispatchers
├── JsonControllers/            invokable JSON endpoints for in-page interactions
├── IndexQueries/               read-side listings/filters/pagination; never mutate
├── ViewModels/                 presentation-ready props; no business rules
├── Resources/                  explicit field allowlists for Inertia/JSON output; never mutate
└── Exports/                    CSV/Excel/PDF output; may use IndexQueries/ViewModels; never mutate
```

- `App/Layer/Dashboard/Booking/*` and `App/Layer/Api/Booking/*` are two interfaces to the same `Core/Domain/Booking` module. Layers mirror domain names exactly but never duplicate domain behavior; route files mirror the same split (`routes/dashboard/booking.php` → labrodev-controller skill).
- Every subfolder above names a playbook class-type concept — nothing else may appear (no domain-local `Support/`; such classes are `Utilities/`).
- `Core/Domain/User/` holds the **mirror User model** — Core's own model on the `users` table — because Core never imports `App\` classes, `App\Models\User` included (→ labrodev-core guideline dependency law).
- There is exactly one Request-less input boundary: controllers map raw input into Spatie Data objects (→ see the labrodev-data skill).

## File header contract (canonical templates)

Naming pattern: namespace mirrors the folder path; class name states intent (→ `labrodev-naming` guideline).

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

## Immutability — why the exclusions exist

The law (`final readonly` where stateless; no `readonly` on Data subclasses, framework-extending classes, static-only classes) is in the `labrodev-core` guideline. The reasons:

- Spatie's base `Data` class is not readonly, and PHP forbids a readonly class extending a non-readonly one — hence `final class BookingData extends Data`.
- Models, Controllers, ViewModels, Resources, and IndexQueries extend non-readonly framework bases — same PHP constraint.
- A static-only class (e.g. a Rule exposing only static methods) has no instance properties, so `readonly` is meaningless — write `final class BookingRule`.
- Mutability and inheritance are intentional decisions, never defaults: if a class is not `final` or not `readonly` where it could be, there must be a stated reason.

## Edge cases

- **Logical vs physical paths**: `Core/...` references are logical modules. Physical homes vary (`src/Core/...`, `packages/*/src/...`, `../core/src/...`). Verify via `composer.json`; apply rules to the logical module either way.
- **Cross-domain needs**: one workflow spanning multiple domains with no natural home goes in `Core/Feature/{FeatureName}` with a small explicit API. When it stabilizes into a business concept, promote it to a proper Domain.
- **Abstract bases**: an abstract class in `Core/Shared` (e.g. `BaseModel`) is the sanctioned exception to `final`. Concrete subclasses are still `final`.
- **Utilities drift**: a domain Utility reused across domains with no domain terminology moves to `Core/Support`.
- **Infrastructure creep**: the moment an Infrastructure adapter makes a business decision, that logic moves to a Domain Service.
- **Legacy zones**: the exemption boundary (`app/Http`, `app/Models`, `app/Actions/Fortify`) is law → `labrodev-core` guideline. When a legacy-zone concept needs real behavior, build it in Core and delegate.
