---
name: labrodev-authorization
description: "Use when creating or reviewing authorization in a Labrodev Laravel project: writing a {Model}Policy class, defining permission constants, wiring #[UsePolicy] on a model, adding #[Authorize] to an invokable controller, or deciding how any endpoint checks who may view/create/update/remove a resource."
license: MIT
metadata:
  author: labrodev
---

# Authorization: Policies, #[UsePolicy], #[Authorize]

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-authorization` guideline** (musts, must-nots); the per-file checklist is `rules/policies.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

Authorization is one cluster with three pieces that must always be wired together:

1. **Policy** — `final class {Model}Policy` in `Core/Domain/{Domain}/Policies/`, answering "may this user perform this action on this object?" in business terms.
2. **Model wiring** — `#[UsePolicy({Model}Policy::class)]` attribute on the model class.
3. **Controller wiring** — class-level `#[Authorize(...)]` attribute on every invokable controller.

## The constant contract (critical)

The permission constant's **value** is the string the Gate uses to find the policy method:

```php
public const string PERMISSION_VIEW = 'view';     // resolves to view()
public const string PERMISSION_UPDATE = 'update'; // resolves to update()
```

`#[Authorize(BookingPolicy::PERMISSION_UPDATE, 'booking')]` only works because `'update'` is a method on `BookingPolicy`. Two distinct string universes exist and must never be mixed:

| String | Universe | Where it lives |
|---|---|---|
| `'view'`, `'update'` | Gate ability = policy method name | Constant values; `#[Authorize]` arguments |
| `'booking.bookings.view'` | Dotted RBAC permission key (e.g. spatie/laravel-permission) | **Inside** method bodies only, via `$user->hasPermissionTo(...)` (optional) |

## Method signature convention

- **Class-level abilities** (list/create — no bound instance): method takes only `?Authenticatable $user` and layers `$user !== null` (plus optional RBAC) → `{Model}Rule::canCreate(...)`.
- **Instance abilities** (update/remove — a bound model): method takes `(?Authenticatable $user, {Model} ${model})` and layers non-null user → ownership/scope → `{Model}Rule::canUpdate(...)` / `canRemove(...)`.

The Policy is where the can-trio is **enforced** — a request that fails a Rule gate dies here with 403, before the Action runs. Invariants that depend on submitted input (which the gate middleware cannot see) stay in the Action as domain-exception guards → see the labrodev-action skill.

## Policy template

Naming pattern: `{Model}Policy` in `Core/Domain/{Domain}/Policies/`. Worked example — Booking domain:

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Policies;

use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Rules\BookingRule;
use Illuminate\Contracts\Auth\Authenticatable;

final class BookingPolicy
{
    public const string PERMISSION_VIEW = 'view';

    public const string PERMISSION_CREATE = 'create';

    public const string PERMISSION_UPDATE = 'update';

    public const string PERMISSION_REMOVE = 'remove';

    public function view(?Authenticatable $user): bool
    {
        return $user !== null;
    }

    public function create(?Authenticatable $user): bool
    {
        return $user !== null
            && BookingRule::canCreate();
    }

    public function update(?Authenticatable $user, Booking $booking): bool
    {
        return $user !== null
            && BookingRule::canUpdate($booking);
    }

    public function remove(?Authenticatable $user, Booking $booking): bool
    {
        return $user !== null
            && BookingRule::canRemove($booking);
    }
}
```

Two project-specific slots exist and are filled without comments (the no-comments law → labrodev-core): with RBAC, `view()` becomes `$user !== null && $user->hasPermissionTo('booking.bookings.view')`; in projects with an ownership model, `update()`/`remove()` add the ownership/scope check between the null check and the Rule gate (e.g. `$user->organisation_id === $booking->organisation_id`).

For the `BookingRule` class itself (what `canCreate`/`canUpdate`/`canRemove` contain) → see the labrodev-action skill.

## Wiring piece 2: #[UsePolicy] on the model

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Models;

use Core\Domain\Booking\Policies\BookingPolicy;
use Core\Shared\Models\BaseModel;
use Illuminate\Database\Eloquent\Attributes\UsePolicy;

#[UsePolicy(BookingPolicy::class)]
final class Booking extends BaseModel
{
    // ...
}
```

This attribute is the **only** policy registration. For everything else about the model (casts(), relations, other attributes) → see the labrodev-model skill.

## Wiring piece 3: #[Authorize] on invokable controllers

Two argument forms, matching `Gate::authorize` semantics:

**Form A — ability + `Model::class`** for endpoints without a bound instance (index, create, store):

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\Controllers;

use App\Http\Controllers\Controller;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Policies\BookingPolicy;
use Illuminate\Routing\Attributes\Controllers\Authorize;

#[Authorize(BookingPolicy::PERMISSION_VIEW, Booking::class)]
final class BookingIndexController extends Controller
{
    // ...
}
```

**Form B — ability + route-parameter name string** for route-model-bound endpoints (show, edit, update, delete). The string is the route parameter name, which matches the `{booking:uuid}` segment in the route definition (UUID binding is the blessed convention):

```php
#[Authorize(BookingPolicy::PERMISSION_UPDATE, 'booking')]
final class BookingUpdateController extends Controller
{
    public function __invoke(
        Booking $booking,
        BookingData $bookingData,
        BookingUpdate $bookingUpdate,
    ): RedirectResponse {
        // ...
    }
}
```

Route: `Route::put('bookings/{booking:uuid}', BookingUpdateController::class)->name('bookings.update');`

The attribute runs the gate as controller middleware, before `__invoke()` — the bound `Booking` instance is passed to `BookingPolicy::update()` automatically. For the rest of the controller body (Data injection, Action invocation, `Inertia::flash` + `to_route()`) → see the labrodev-controller skill.

## Edge cases

**Runtime-setup exception.** When the subject is only known *after* runtime setup — e.g. the controller first resolves a `ConfigurationSet` via a fetcher service, with no route-model binding available — the class-level attribute cannot express the check. Authorize in the method body instead, immediately after the subject is resolved:

```php
use Illuminate\Support\Facades\Gate;

final class ConfigurationSetEditController extends Controller
{
    public function __invoke(ConfigurationSetFetcher $configurationSetFetcher): Response
    {
        $configurationSet = $configurationSetFetcher();

        Gate::authorize(ConfigurationSetPolicy::PERMISSION_UPDATE, $configurationSet);

        // ...
    }
}
```

This is the **only** accepted reason to skip `#[Authorize]`. If the subject is bindable via `{model:uuid}`, use Form B instead.

**Cross-domain policy reuse.** A Layer controller in one domain module may authorize against another domain's policy (e.g. an invoice endpoint checking `BookingPolicy::PERMISSION_VIEW` on a bound booking). This is allowed — policies are Core classes, and App/Layer → Core dependency direction permits it. Do not duplicate the policy in the second domain.

**Nullable user.** Signatures take `?Authenticatable` deliberately: policy methods run for guests too, and each method makes the `$user !== null` check explicit rather than relying on framework guest-denial magic.

**Non-CRUD abilities.** Custom endpoints get custom permissions following the same contract: `public const string PERMISSION_CANCEL = 'cancel';` with a `cancel(?Authenticatable $user, Booking $booking): bool` method. Never overload an existing ability with unrelated meaning.
