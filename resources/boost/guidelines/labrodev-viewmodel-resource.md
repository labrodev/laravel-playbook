# Labrodev ViewModels & Resources

Always-on law. These two classes form the presentation boundary between the database and the browser: the **ViewModel** decides *which* data reaches the page, the **Resource** decides *what fields* of it are exposed (an explicit allowlist). Neither may be bypassed — raw Eloquent models never reach Inertia or JSON output. Anatomy and templates → the `labrodev-viewmodel-resource` skill; per-file checks → `rules/viewmodels-resources.md`.

## Musts

### ViewModels

- Live in `App/Layer/{Layer}/{Domain}/ViewModels`; `final`, extending `Spatie\ViewModels\ViewModel` directly; named `{Model}{Context}ViewModel` (`BookingIndexViewModel`) (→ labrodev-naming).
- **Always override `toArray()`** returning `array<string, mixed>` — the controller passes `$viewModel->toArray()` to `Inertia::render(...)`; Spatie's magic view-data features are not used.
- The **primary subject is constructor-injected** as `private readonly` — the paginator from the controller's IndexQuery, or the route-bound model.
- **Collecting additional page data is the ViewModel's task, not the controller's**: select options, auxiliary collections, flags, counts needed by the template or rendering logic are fetched inside the ViewModel through Query classes (`resolve({Model}Query::class)`) — never by piling extra query injections into the controller (→ labrodev-controller), and never via inline `Model::query()` (→ labrodev-query).
- Resources are **always materialized with `->resolve()`** inside the ViewModel — never returned as `JsonResource` instances to Inertia.
- Paginated collections use the canonical **pagination envelope**: `data`, `current_page`, `last_page`, `per_page`, `total`, `from`, `to`, `links`.
- Select/filter option props originate in the ViewModel — the frontend never hardcodes option values; enum options emit value + label via EnumMapper (→ labrodev-enum).
- Show/edit ViewModels load required relations (`$this->model->load([...])`) inside `toArray()` before resolving the Resource.

### Resources

- Live in `App/Layer/{Layer}/{Domain}/Resources` — Resources are delivery-surface classes, **never under `Core/Domain`**; `final`, extending `Illuminate\Http\Resources\Json\JsonResource`, with a `JsonResource<Model>` class docblock; named `{Model}{Context}Resource` — one model may have multiple Resources per context.
- `toArray()` is an **explicit allowlist**: only fields the specific page/consumer actually renders — every field must have a corresponding UI element.
- **`uuid` is the public identifier** — expose `uuid`, never the internal integer `id` (unless the page explicitly displays it, which is almost never).
- Include the **`fetchModel()` typed guard**: verify `$this->resource instanceof Model` or throw `ObjectMissed::make(Model::class)`; all field access goes through the guarded model.
- Field grammar: snake_case keys; datetimes `->toDateTimeString()`, dates `->toDateString()`; enum fields as `value` + `*_label` pairs (→ labrodev-enum).
- May compute derived presentation values (formatting, totals, concatenated names).

## Must-nots

### ViewModels

- No business logic, no mutations, no Action/Service/Job calls — presentation shaping and mapping only.
- No raw models, `$model->toArray()`, or unfiltered collections in the returned array — everything goes through a Resource.
- No UI translation payloads for Inertia pages — copy is handled by frontend i18n (→ labrodev-inertia-react).
- Domain code (Actions, Services, Queries) never constructs or returns ViewModels or Resources — they are instantiated at the delivery boundary (controller/ViewModel).

### Resources

- Must not mutate state or trigger workflows (Actions, Jobs).
- Must not spread `$model->toArray()` / `attributesToArray()` into the output — that exposes every attribute and loaded relation.
- Must not expose: internal integer ids, password hashes, tokens/secrets, internal system fields (`created_by`, internal flags) unless explicitly displayed, pivot/debug metadata.
