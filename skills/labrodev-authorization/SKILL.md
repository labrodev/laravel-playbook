---
name: labrodev-authorization
description: "Use when creating or reviewing authorization in a Labrodev Laravel project: writing a {Model}Policy class, defining permission constants, wiring #[UsePolicy] on a model, adding #[Authorize] to an invokable controller, or deciding how any endpoint checks who may view/create/update/remove a resource."
license: MIT
metadata:
  author: labrodev
---

# Authorization: Policies, #[UsePolicy], #[Authorize]

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

Authorization is one cluster with three pieces that must always be wired together:

1. **Policy** — `final class {Model}Policy` in `Core/Domain/{Domain}/Policies/`, answering "may this user perform this action on this object?" in business terms.
2. **Model wiring** — `#[UsePolicy({Model}Policy::class)]` attribute on the model class.
3. **Controller wiring** — class-level `#[Authorize(...)]` attribute on every invokable controller.

## Musts

- Policies live in `Core/Domain/{Domain}/Policies/` and are named `{Model}Policy` (e.g. `BookingPolicy`).
- Every permission is declared as a `public const string` on the policy, and **the constant VALUE must equal the policy METHOD name** — see "The constant contract" below. This is non-negotiable.
- Attach the policy to its model with `#[UsePolicy({Model}Policy::class)]` (`Illuminate\Database\Eloquent\Attributes\UsePolicy`) on the model class.
- Every invokable controller carries a class-level `#[Authorize(...)]` attribute (`Illuminate\Routing\Attributes\Controllers\Authorize`) so the gate runs as controller middleware — this applies to Inertia controllers **and** JsonControllers alike.
- Policy methods layer checks in a fixed order: **non-null user → ownership/scope → domain `{Model}Rule` gate**.
- Object-state gates (`isEditable`, `canBeDeleted`) are delegated to the domain Rule class, called statically and directly: `BookingRule::canBeDeleted($booking)` — never via a helper on the model.
- Define **only** the permissions that exist for the model's real use cases. The set is not fixed; it is not always view/create/update/remove (an append-only log model may have only `PERMISSION_VIEW` and `PERMISSION_CREATE`).
- **Every endpoint must be covered.** A controller without `#[Authorize]` (and without the documented runtime-setup exception) is a review blocker, not a style nit.

## Must-nots

- Never register policies via `Gate::policy(...)` in a service provider. `#[UsePolicy]` on the model is the only wiring mechanism.
- Never use a dotted RBAC key (e.g. `'booking.bookings.view'`) as a permission constant value — a dotted value cannot resolve to a policy method through the Gate. Dotted keys live **inside** method bodies only (see below).
- Policies must not mutate state, implement workflows, call Actions/Jobs, or duplicate business logic already expressed in Rules.
- Never call `$this->authorize(...)` or `Gate::authorize(...)` inside `__invoke()` when the subject is known up front — use the class-level attribute. The in-body form is reserved for the runtime-setup exception.
- The ownership/scope check is a project-specific slot (shown commented in the template) — the playbook policy template itself carries no app-specific scoping logic.

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

- **Class-level abilities** (list/create — no bound instance): method takes only `?Authenticatable $user` and typically checks `$user !== null` (plus optional RBAC).
- **Instance abilities** (update/remove — a bound model): method takes `(?Authenticatable $user, {Model} ${model})` and layers non-null user → ownership/scope → `{Model}Rule` gate.

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
        // With RBAC: return $user !== null && $user->hasPermissionTo('booking.bookings.view');
    }

    public function create(?Authenticatable $user): bool
    {
        return $user !== null;
    }

    public function update(?Authenticatable $user, Booking $booking): bool
    {
        return $user !== null
            // Ownership / scope check (project-specific — adjust to the project's ownership model):
            // && $user->organisation_id === $booking->organisation_id
            && BookingRule::isEditable($booking);
    }

    public function remove(?Authenticatable $user, Booking $booking): bool
    {
        return $user !== null
            && BookingRule::canBeDeleted($booking);
    }
}
```

For the `BookingRule` class itself (what `isEditable`/`canBeDeleted` contain) → see the labrodev-action skill.

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

## Review checklist

- Does every permission constant's VALUE exactly equal a policy method name (no dotted strings as constant values)?
- Is the policy `final`, in `Core/Domain/{Domain}/Policies/`, named `{Model}Policy`, with `declare(strict_types=1)`?
- Is the policy attached via `#[UsePolicy(...)]` on the model — and is there no `Gate::policy()` call anywhere in a provider?
- Does every invokable controller (including JsonControllers) carry class-level `#[Authorize(...)]`, or a documented runtime-setup `Gate::authorize` in the body?
- Do instance-ability attributes use the route-parameter name string that matches the `{model:uuid}` route segment?
- Do update/remove methods layer checks as: non-null user → ownership/scope → `{Model}Rule` gate?
- Are object-state gates delegated to `{Model}Rule::...` (called statically, not via the model) instead of being re-implemented in the policy?
- Is the policy free of mutations, workflows, and calls to Actions/Jobs?
- Are only the permissions that real use cases need defined (no reflexive view/create/update/remove boilerplate)?
