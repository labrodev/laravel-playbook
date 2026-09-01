---
paths:
  - app/Layer/**/Controllers/**
  - app/Layer/**/JsonControllers/**
  - routes/**
---

# Controllers & routes — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular; the class body contains no comments — typed PHPStan annotations only.
- Every endpoint is a `final` invokable controller with a single `__invoke()` — no multi-method controllers.
- Authorization is a class-level `#[Authorize(...)]` attribute on EVERY controller (runtime-setup exception → labrodev-authorization).
- The authenticated user arrives via `#[CurrentUser] User $user` on `__invoke()` — no `auth()->user()`, `request()->user()`, or `Auth::user()` in the body.
- No business logic: no branching, calculations, inline queries, model mutations, no `DB::transaction()`.
- The controller passes only the primary subject (IndexQuery result or bound model) — no extra query injections gathering auxiliary page data; that collection lives in the ViewModel (→ rules/viewmodels-resources.md).
- Write endpoints inject a Spatie Data object + Core Action, invoked as a callable with named arguments — no Request classes, no `validate()`.
- Inertia write endpoints end with `Inertia::flash('toast', ...)` + `to_route(...)` — never `redirect()->route()`.
- Read endpoints take input only from route-model binding + the query string; return `Inertia::render` with `$viewModel->toArray()`.
- JsonControllers and API controllers return `JsonResource`/`AnonymousResourceCollection`, never raw arrays; JsonControllers live under a `json/` prefix and authorize like any controller.
- Routes are explicit (no `Route::resource()`, no `[Controller::class, 'method']` arrays), kebab-case URLs, dot names, `{model:uuid}` binding; removals are POST `/remove` to dedicated `*RemoveController` classes.
- Route files mirror the Layer/Domain structure (`routes/{layer}/{domain}.php`); `App/Layer/{Layer}/{Domain}` folder names mirror `Core/Domain/{Domain}` exactly.
- Literal segments (`bookings/export`, `bookings/create`) are registered before `{booking:uuid}` routes.

Full anatomy and templates → labrodev-controller skill.
