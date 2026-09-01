# Labrodev Controllers & Routes

Always-on law. A controller is a **thin dispatcher**: it receives HTTP input, delegates to Core, and returns a response — nothing else. Controllers live in `App/Layer/{Layer}/{Domain}/Controllers` (Inertia/API) and `JsonControllers` (in-page JSON). Anatomy and templates → the `labrodev-controller` skill; per-file checks → `rules/controllers.md`.

## Musts

- **One HTTP endpoint = one invokable controller class** with a single `__invoke()`, extending `App\Http\Controllers\Controller`. CRUD flows use a family of per-action classes (`BookingIndexController`, `BookingStoreController`, ...).
- **Read controllers** inject a Layer IndexQuery (listings) or receive a route-model-bound model, build a ViewModel, and return `Inertia::render('kebab-view', $viewModel->toArray())`.
- **Controllers pass the primary subject only** — the IndexQuery result or the bound model. Any additional data the page needs (select options, auxiliary collections, flags, counts) is collected by the ViewModel itself (→ labrodev-viewmodel-resource). A controller injecting several queries just to feed the ViewModel is the smell.
- **Write controllers** inject a Spatie Data object (validation happens on resolve) and a Core Action into `__invoke()`, invoke the Action as a callable with named arguments, flash a toast, and redirect with `to_route()`.
- **Every controller is gated with a class-level `#[Authorize(...)]` attribute** so the gate runs as controller middleware — no exceptions beyond the documented runtime-setup case (→ labrodev-authorization).
- **The authenticated user enters via the `#[CurrentUser]` parameter attribute** (`Illuminate\Container\Attributes\CurrentUser`) on `__invoke()` — `#[CurrentUser] User $user` — never `auth()->user()`, `request()->user()`, or `Auth::user()` in the body.
- **Actions exist only in Core** — controllers map input into Data objects and delegate every mutation to a Core Action (or a pipeline-orchestrating Service → labrodev-pipeline). Never create `App/Layer/…/Actions/`.
- **Read endpoints take input from route-model binding and the query string** (`request()->integer(...)`, `request()->string(...)`) — no Request classes, no Data classes, no validation on reads.
- **Route-model binding is always by uuid**: `{booking:uuid}`, never by id.

## Route musts

- **Route files mirror the Layer/Domain structure**: one file per layer+domain — `routes/{layer}/{domain}.php` (kebab-case, e.g. `routes/dashboard/booking.php`, `routes/api/booking.php`) — holding only that domain's routes for that layer.
- **`App/Layer/{Layer}/{Domain}` mirrors `Core/Domain/{Domain}`** — the same singular domain noun on both sides; a Layer never invents a domain name Core doesn't have.
- Routes are **explicit** — register the invokable class directly: `Route::get('bookings', BookingIndexController::class)`.
- Kebab-case URLs, dot-notation route names.
- Domain removals/archiving use an explicit **POST `/remove`** route, not HTTP DELETE.
- Access control via `Route::middleware([...])->group(...)` — the stack is contextual to the delivery layer.
- JsonController routes nest under a `json/` prefix inside the resource route group, same `web` middleware (session + CSRF).

## Must-nots

- No business logic in controllers: no domain branching, no calculations, no inline Eloquent queries, no `->save()` / `->update()` / `::create()`, no `DB::transaction()`.
- No data-gathering in controllers beyond the primary subject — auxiliary page data is the ViewModel's job, fetched there through Query classes (→ labrodev-viewmodel-resource).
- No `auth()->user()` / `request()->user()` / `Auth::user()` in controller bodies — the user is a `#[CurrentUser]` parameter.
- No Request classes, ever. No `->validate()` or `Validator::make()` — validation lives in Data classes (→ labrodev-data).
- No `Route::resource()`. No multi-method controllers, ever. No `[Controller::class, 'method']` route registrations.
- No `redirect()->route()` in Inertia write controllers — use `to_route()`. No `session()->flash()` for toasts — use `Inertia::flash('toast', ...)`.
- No raw arrays or `response()->json([...])` from JsonControllers or API controllers — return `JsonResource` / `AnonymousResourceCollection` only.
- No skipped authorization on JSON endpoints.
- No `$this->authorize(...)` — the Laravel 13 base `Controller` has no `authorize()` helper.
