---
name: labrodev-controller
description: "Use when creating, reviewing, or wiring HTTP controllers or routes in a Labrodev Laravel project — invokable Inertia controllers (BookingIndexController, BookingStoreController), JsonControllers under a json/ prefix, API controllers, or route files (explicit routes, {model:uuid} binding, POST /remove)."
license: MIT
metadata:
  author: labrodev
---

# Controllers and Routes

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-controller` guideline** (musts, must-nots, route rules); the per-file checklist is `rules/controllers.md`. This skill holds the craft: controller anatomy, canonical templates, and edge cases.

Controllers live in `App/Layer/<Layer>/{Domain}/Controllers` (Inertia/API) and `App/Layer/<Layer>/{Domain}/JsonControllers` (in-page JSON) — the `{Domain}` folder name mirrors `Core/Domain/{Domain}` exactly. A controller is a **thin dispatcher**: it receives HTTP input, delegates to Core, and returns a response. Nothing else — collecting page data beyond the primary subject is the ViewModel's job (→ labrodev-viewmodel-resource skill).

## The current user

When an endpoint needs the authenticated user, it arrives as a parameter via Laravel's container attribute — never fetched in the body:

```php
use App\Models\User;
use Illuminate\Container\Attributes\CurrentUser;

public function __invoke(
    #[CurrentUser] User $user,
    BookingData $bookingData,
    BookingCreate $bookingCreate,
): RedirectResponse {
    // ...
}
```

`auth()->user()`, `request()->user()`, and `Auth::user()` in a controller body (with their `/** @var User $user */` docblock crutches) are the anti-pattern this replaces.

## Read controller (Inertia)

Naming: `{Model}{Action}Controller` — here `Booking` + `Index` + `Controller`.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\Controllers;

use App\Http\Controllers\Controller;
use App\Layer\Dashboard\Booking\IndexQueries\BookingIndexQuery;
use App\Layer\Dashboard\Booking\ViewModels\BookingIndexViewModel;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Policies\BookingPolicy;
use Illuminate\Routing\Attributes\Controllers\Authorize;
use Inertia\Inertia;
use Inertia\Response;

#[Authorize(BookingPolicy::PERMISSION_VIEW, Booking::class)]
final class BookingIndexController extends Controller
{
    public function __invoke(BookingIndexQuery $bookingIndexQuery): Response
    {
        $items = $bookingIndexQuery
            ->paginate(request()->integer('per_page', 50))
            ->withQueryString();

        $viewModel = new BookingIndexViewModel(items: $items);

        return Inertia::render('bookings/index', $viewModel->toArray());
    }
}
```

IndexQuery internals → see the labrodev-query skill. ViewModel internals → see the labrodev-viewmodel-resource skill.

The controller hands over **only the primary subject** (here the paginator; on show/edit pages the bound model). If the page also needs select options, auxiliary lists, or flags, the ViewModel collects them itself through Query classes — the controller does not grow extra query injections to feed it.

## Write controller (Inertia)

Naming: `{Model}{Action}Controller` — here `BookingStoreController`. Flow: Layer Controller → Spatie Data → Core Action → Model.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\Controllers;

use App\Http\Controllers\Controller;
use Core\Domain\Booking\Actions\BookingCreate;
use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Policies\BookingPolicy;
use Illuminate\Http\RedirectResponse;
use Illuminate\Routing\Attributes\Controllers\Authorize;
use Inertia\Inertia;

#[Authorize(BookingPolicy::PERMISSION_CREATE, Booking::class)]
final class BookingStoreController extends Controller
{
    public function __invoke(
        BookingData $bookingData,
        BookingCreate $bookingCreate,
    ): RedirectResponse {
        $bookingCreate(
            bookingData: $bookingData,
        );

        Inertia::flash('toast', ['type' => 'success', 'message' => trans('Booking created.')]);

        return to_route('bookings.index');
    }
}
```

Response idiom (fixed): `Inertia::flash('toast', ['type' => 'success|error|info', 'message' => trans(...)])` then `return to_route(...)`.

**Update variant:** add the route-model-bound model parameter first, switch the attribute to `#[Authorize(BookingPolicy::PERMISSION_UPDATE, 'booking')]`, and invoke `$bookingUpdate(booking: $booking, bookingData: $bookingData);`.

## JsonControllers (in-page JSON)

For in-page interactions that must not trigger a full Inertia visit: wizard steps, autocomplete/search, dynamic server-side calculations. When the result is a page navigation or form submission, use Inertia instead — JsonControllers are never for initial page data loading.

Naming: `{Entity}{Verb}Controller` where the verb describes the operation: `Get`, `Find`, `Calculate`, `Search` (e.g. `BookingSearchController`, `ServiceGetController`).

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\JsonControllers;

use App\Http\Controllers\Controller;
use App\Layer\Dashboard\Booking\Resources\BookingResource;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Policies\BookingPolicy;
use Core\Domain\Booking\Services\BookingSearcher;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Routing\Attributes\Controllers\Authorize;

#[Authorize(BookingPolicy::PERMISSION_VIEW, Booking::class)]
final class BookingSearchController extends Controller
{
    public function __invoke(BookingSearcher $bookingSearcher): AnonymousResourceCollection
    {
        $results = $bookingSearcher(term: request()->string('term')->toString());

        return BookingResource::collection($results);
    }
}
```

JsonController routes are nested under a `json/` prefix inside the resource route group and run under the same `web` middleware group (session + CSRF) — see the routes template below. The frontend calls them via `@/lib/http` helpers, never raw `fetch()`/`axios` → see the labrodev-inertia-react skill. Resource internals → see the labrodev-viewmodel-resource skill.

## API controller variant

Same anatomy — invokable, thin, class-level attribute — but lives under `App/Layer/Api/{Domain}/Controllers` and returns its Layer's Resources instead of Inertia responses. Actions for atomic mutations; pipeline-orchestrating Services for staged workflows → see the labrodev-pipeline skill.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Api\Booking\Controllers;

use App\Http\Controllers\Controller;
use App\Layer\Api\Booking\Resources\BookingResource;
use Core\Domain\Booking\Data\BookingData;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Services\BookingRegistrationService;
use Core\Domain\Booking\Policies\BookingPolicy;
use Illuminate\Http\JsonResponse;
use Illuminate\Routing\Attributes\Controllers\Authorize;

// WRITE: mutation endpoint
#[Authorize(BookingPolicy::PERMISSION_CREATE, Booking::class)]
final class BookingCreateController extends Controller
{
    public function __invoke(
        BookingData $bookingData,
        BookingRegistrationService $bookingRegistrationService,
    ): JsonResponse {
        $result = $bookingRegistrationService(
            bookingData: $bookingData,
        );

        return BookingResource::make($result)
            ->response()
            ->setStatusCode(201);
    }
}

// READ: query endpoint — input via {booking:uuid} binding + query string only.
#[Authorize(BookingPolicy::PERMISSION_VIEW, 'booking')]
final class BookingShowController extends Controller
{
    public function __invoke(Booking $booking): BookingResource
    {
        return BookingResource::make($booking);
    }
}
```

## Routes (canonical template)

Route law (explicit routes, kebab-case, `{model:uuid}`, POST `/remove`) → `labrodev-controller` guideline. Route files mirror the Layer/Domain structure — this template is `routes/dashboard/booking.php`, holding only the Dashboard layer's Booking routes; the layer's route files are loaded from `bootstrap/app.php` (or the layer's route service registration).

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Route;

Route::middleware(['web', 'auth', 'verified'])->group(function () {

    Route::prefix('bookings')->name('bookings.')->group(function () {
        Route::get('/', BookingIndexController::class)->name('index');
        Route::get('export', BookingExportController::class)->name('export'); // before {booking:uuid}
        Route::get('create', BookingCreateController::class)->name('create');
        Route::post('/', BookingStoreController::class)->name('store');
        Route::get('{booking:uuid}', BookingShowController::class)->name('show');
        Route::get('{booking:uuid}/edit', BookingEditController::class)->name('edit');
        Route::put('{booking:uuid}', BookingUpdateController::class)->name('update');
        Route::post('{booking:uuid}/remove', BookingRemoveController::class)->name('remove');

        // In-page JSON endpoints — same session/CSRF middleware, json/ prefix
        Route::prefix('json')->name('json.')->group(function () {
            Route::get('search', BookingSearchController::class)->name('search');
        });
    });

    // Nested / relation routes stay explicit:
    // Route::post('projects/{project:uuid}/members', ProjectMemberCreateController::class)
    //     ->name('projects.members.create');
    // Route::post('projects/{project:uuid}/members/{member:uuid}/remove', ProjectMemberRemoveController::class)
    //     ->name('projects.members.remove');
});
```

## Edge cases

- **Literal segments before uuid bindings.** Register fixed-path routes (`bookings/export`, `bookings/create`) before `bookings/{booking:uuid}` so the binding never swallows them.
- **Subject known only after runtime setup** → see the labrodev-authorization skill (runtime-setup exception).
- **Custom state-changing actions.** Use POST for non-idempotent commands (`bookings/{booking:uuid}/toggle`), PUT for idempotent sub-resource updates (`bookings/{booking:uuid}/notes`).
- **Batch form + submission on one endpoint.** Use `Route::match(['GET', 'POST'], 'bookings/batch-approve', BookingBatchApproveController::class)`.
- **Exports.** An export controller is a read controller: inject the IndexQuery, hand it to the Export class, return the download response. Never mutate state.
- **PaginatedIndex reads.** Always `paginate(request()->integer('per_page', 50))->withQueryString()` so filters survive pagination.
