---
name: labrodev-controller
description: "Use when creating, reviewing, or wiring HTTP controllers or routes in a Labrodev Laravel project — invokable Inertia controllers (BookingIndexController, BookingStoreController), JsonControllers under a json/ prefix, API controllers, Blade legacy controllers, or route files (explicit routes, {model:uuid} binding, POST /remove)."
license: MIT
metadata:
  author: labrodev
---

# Controllers and Routes

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

Controllers live in `App/Layer/<Layer>/{Domain}/Controllers` (Inertia/Blade/API) and `App/Layer/<Layer>/{Domain}/JsonControllers` (in-page JSON). A controller is a **thin dispatcher**: it receives HTTP input, delegates to Core, and returns a response. Nothing else.

## Rules

### Musts

- **One HTTP endpoint = one invokable controller class** with a single `__invoke()` method, extending `App\Http\Controllers\Controller`. CRUD flows use a family of per-action classes: `BookingIndexController`, `BookingCreateController`, `BookingStoreController`, `BookingShowController`, `BookingEditController`, `BookingUpdateController`, `BookingDeleteController`.
- **Read controllers** inject a Layer IndexQuery (listings) or receive a route-model-bound model, build a ViewModel, and return `Inertia::render('kebab-view', $viewModel->toArray())`.
- **Write controllers** inject a Spatie Data object (validation happens on resolve) and a Core Action into `__invoke()`, invoke the Action as a callable with named arguments, flash a toast, and redirect with `to_route()`.
- **Authorization goes on the class** as a class-level `#[Authorize(...)]` attribute (`Illuminate\Routing\Attributes\Controllers\Authorize`) so the gate runs as controller middleware — argument forms and the policy contract → see the labrodev-authorization skill.
- **Actions exist only in Core.** Controllers map input into Data objects and delegate every mutation to a Core Action or Orchestrator. Never create `App/Layer/…/Actions/`.
- **Read endpoints take input from route-model binding and the query string** (`request()->integer(...)`, `request()->string(...)`). No Request classes, no Data classes, no validation on reads → see the labrodev-data skill.
- **Route-model binding is always by uuid**: `{booking:uuid}`, never by id.
- **File header contract** (strict types, final) → see the labrodev-core skill.

### Must-nots

- No business logic in controllers: no domain branching, no calculations, no inline Eloquent queries, no `->save()` / `->update()` / `::create()` calls, no `DB::transaction()`. All of that belongs in Core → see the labrodev-action skill.
- No Request classes, ever. No `->validate()` or `Validator::make()` in controllers — validation lives in Data classes → see the labrodev-data skill.
- No `Route::resource()`. No multi-method controllers on Inertia/Dashboard/API surfaces (Blade legacy is the sole exception, below).
- No `redirect()->route()` in Inertia write controllers — use `to_route()`. No `session()->flash()` for toasts — use `Inertia::flash('toast', ...)`.
- No raw arrays or `response()->json([...])` from JsonControllers or API controllers — return `JsonResource` / `AnonymousResourceCollection` only.
- No skipped authorization on JSON endpoints — JsonControllers authorize exactly like other controllers.
- No `$this->authorize(...)` — the Laravel 13 base `Controller` has no `authorize()` helper.

Class, method, and variable naming rules (including the `BookingCreateData $bookingCreateData` mirror rule and callable invocation style) → see the labrodev-naming skill.

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

## Write controller (Inertia)

Naming: `{Model}{Action}Controller` — here `BookingStoreController`. Flow: Layer Controller → Spatie Data → Core Action → Model.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\Controllers;

use App\Http\Controllers\Controller;
use Core\Domain\Booking\Actions\BookingCreate;
use Core\Domain\Booking\Data\BookingCreateData;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Policies\BookingPolicy;
use Illuminate\Http\RedirectResponse;
use Illuminate\Routing\Attributes\Controllers\Authorize;
use Inertia\Inertia;

#[Authorize(BookingPolicy::PERMISSION_CREATE, Booking::class)]
final class BookingStoreController extends Controller
{
    public function __invoke(
        BookingCreateData $bookingCreateData,
        BookingCreate $bookingCreate,
    ): RedirectResponse {
        $bookingCreate(
            bookingCreateData: $bookingCreateData,
        );

        Inertia::flash('toast', ['type' => 'success', 'message' => trans('Booking created.')]);

        return to_route('bookings.index');
    }
}
```

Response idiom (fixed): `Inertia::flash('toast', ['type' => 'success|error|info', 'message' => trans(...)])` then `return to_route(...)`.

**Update variant:** add the route-model-bound model parameter first, switch the attribute to `#[Authorize(BookingPolicy::PERMISSION_UPDATE, 'booking')]`, and invoke `$bookingUpdate(booking: $booking, bookingUpdateData: $bookingUpdateData);`.

## JsonControllers (in-page JSON)

For in-page interactions that must not trigger a full Inertia visit: wizard steps, autocomplete/search, dynamic server-side calculations. When the result is a page navigation or form submission, use Inertia instead — JsonControllers are never for initial page data loading.

Naming: `{Entity}{Verb}Controller` where the verb describes the operation: `Get`, `Find`, `Calculate`, `Search` (e.g. `BookingSearchController`, `ServiceGetController`).

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\JsonControllers;

use App\Http\Controllers\Controller;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Policies\BookingPolicy;
use Core\Domain\Booking\Resources\BookingResource;
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

Same anatomy — invokable, thin, class-level attribute — but lives under `App/Layer/Api/{Domain}/Controllers` and returns Domain Resources instead of Inertia responses. Prefer Orchestrators for complex write flows, Actions for atomic mutations.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Api\Booking\Controllers;

use App\Http\Controllers\Controller;
use Core\Domain\Booking\Data\BookingCreateData;
use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Orchestrators\BookingCreateOrchestrator;
use Core\Domain\Booking\Policies\BookingPolicy;
use Core\Domain\Booking\Resources\BookingResource;
use Illuminate\Http\JsonResponse;
use Illuminate\Routing\Attributes\Controllers\Authorize;

// WRITE: mutation endpoint
#[Authorize(BookingPolicy::PERMISSION_CREATE, Booking::class)]
final class BookingCreateController extends Controller
{
    public function __invoke(
        BookingCreateData $bookingCreateData,
        BookingCreateOrchestrator $bookingCreateOrchestrator,
    ): JsonResponse {
        // Orchestrator entry point is execute(); its parameter is named $input.
        $result = $bookingCreateOrchestrator->execute(
            input: $bookingCreateData,
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

## Blade multi-method controller (legacy exception)

Blade surfaces are the **only** place where one class may hold multiple actions. Because multiple methods share one class, class-level `#[Authorize]` cannot express per-action abilities — authorize per method with `Gate::authorize(...)` (the base `Controller` has no `authorize()` helper). Everything else stays identical: Data objects for input, Core Actions invoked as callables, ViewModels for output.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\Controllers;

use App\Http\Controllers\Controller;
use App\Layer\Dashboard\Booking\IndexQueries\BookingIndexQuery;
use App\Layer\Dashboard\Booking\ViewModels\BookingIndexViewModel;
use Core\Domain\Booking\Actions\BookingCreate;
use Core\Domain\Booking\Data\BookingCreateData;
use Core\Domain\Booking\Models\Booking;
use Illuminate\Contracts\View\View;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Gate;

final class BookingController extends Controller
{
    public function __invoke(BookingIndexQuery $bookingIndexQuery): View
    {
        Gate::authorize('view', Booking::class);

        $items = $bookingIndexQuery
            ->paginate(request()->integer('per_page', 50))
            ->withQueryString();

        $viewModel = new BookingIndexViewModel(items: $items);

        return view('bookings.index', $viewModel->toArray());
    }

    public function store(BookingCreateData $bookingCreateData, BookingCreate $bookingCreate): RedirectResponse
    {
        Gate::authorize('create', Booking::class);

        $bookingCreate(bookingCreateData: $bookingCreateData);

        return redirect()->route('bookings.index')->with('success', __('Created.'));
    }

    public function remove(Booking $booking /*, BookingRemove $bookingRemove */): RedirectResponse
    {
        Gate::authorize('remove', $booking);

        // $bookingRemove(booking: $booking);

        return redirect()->route('bookings.index')->with('success', __('Removed.'));
    }
}
```

Never introduce this shape for new Inertia/Dashboard/API endpoints. Vendor/starter code exemptions → see the labrodev-core skill.

## Routes

Rules:

- Routes are **explicit** — no `Route::resource()`.
- One invokable controller class per route; register the class directly: `Route::get('bookings', BookingIndexController::class)`.
- Kebab-case URLs, dot-notation route names.
- Route-model binding always `{model:uuid}`, never by id.
- Domain removals/archiving use an explicit **POST `/remove`** route, not HTTP DELETE.
- Access control via `Route::middleware([...])->group(...)`; the middleware stack is contextual to the delivery layer (auth/verified/api throttling/etc.).
- Multi-method `[Controller::class, 'method']` registrations are the Blade legacy exception only.

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
## Review checklist

1. Is every Inertia/Dashboard/API endpoint a `final` invokable controller with a single `__invoke()`?
2. Is authorization declared as a class-level `#[Authorize(...)]` attribute (or `Gate::authorize` only in the documented exceptions)?
3. Is the controller free of business logic — no branching, calculations, inline queries, model mutations, or transactions?
4. Do write endpoints inject a Spatie Data object and a Core Action (invoked as a callable with named arguments), with no Request classes or `validate()` calls?
5. Do Inertia write endpoints end with `Inertia::flash('toast', ...)` + `to_route(...)` (not `redirect()->route()`)?
6. Do read endpoints take input only from route-model binding and the query string, returning `Inertia::render` with `$viewModel->toArray()`?
7. Do JsonControllers and API controllers return `JsonResource`/`AnonymousResourceCollection`, never raw arrays, and sit under a `json/` prefix (JsonControllers)?
8. Are all routes explicit (no `Route::resource()`), with kebab-case URLs, dot names, and `{model:uuid}` binding?
9. Are removals wired as POST `/remove` routes to dedicated `*RemoveController` classes?
10. Are multi-method controllers absent everywhere except legacy Blade surfaces?
