---
paths:
  - app/Layer/**/Controllers/**
  - app/Layer/**/JsonControllers/**
  - routes/**
---

# Controllers & routes — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular.
- Every endpoint is a `final` invokable controller with a single `__invoke()` — no multi-method controllers.
- Authorization is a class-level `#[Authorize(...)]` attribute (runtime-setup exception → labrodev-authorization).
- No business logic: no branching, calculations, inline queries, model mutations, no `DB::transaction()`.
- Write endpoints inject a Spatie Data object + Core Action, invoked as a callable with named arguments — no Request classes, no `validate()`.
- Inertia write endpoints end with `Inertia::flash('toast', ...)` + `to_route(...)` — never `redirect()->route()`.
- Read endpoints take input only from route-model binding + the query string; return `Inertia::render` with `$viewModel->toArray()`.
- JsonControllers and API controllers return `JsonResource`/`AnonymousResourceCollection`, never raw arrays; JsonControllers live under a `json/` prefix and authorize like any controller.
- Routes are explicit (no `Route::resource()`, no `[Controller::class, 'method']` arrays), kebab-case URLs, dot names, `{model:uuid}` binding; removals are POST `/remove` to dedicated `*RemoveController` classes.
- Literal segments (`bookings/export`, `bookings/create`) are registered before `{booking:uuid}` routes.

Full anatomy and templates → labrodev-controller skill.
