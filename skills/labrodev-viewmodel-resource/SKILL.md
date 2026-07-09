---
name: labrodev-viewmodel-resource
description: "Use when creating, reviewing, or naming ViewModels (App/Layer/*/ViewModels, e.g. BookingIndexViewModel, BookingShowViewModel) or Resources (Core/Domain/*/Resources, e.g. BookingIndexResource) — i.e. whenever data is shaped for an Inertia page, Blade template, or JSON payload, or when deciding which fields a model may expose to the browser."
license: MIT
metadata:
  author: labrodev
---

# ViewModels & Resources — the presentation boundary

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

Together these two classes form the security boundary between the database and the browser:

- The **ViewModel** decides *which* data reaches the page (which models, which relations, which options).
- The **Resource** decides *what fields* of that data are exposed (an explicit allowlist).

Neither may be bypassed. Raw Eloquent models never reach Inertia or JSON output.

## Responsibility split (controller vs ViewModel vs Resource)

| Concern | Owner |
|---|---|
| Authorize, run IndexQuery, paginate, construct ViewModel, `Inertia::render(...)` | Controller → see the labrodev-controller skill |
| Collect/shape all props for the page: pagination envelope, resolved Resources, select options | ViewModel (`App/Layer/{Layer}/{Domain}/ViewModels`) |
| Allowlist mapping of a single model to public fields | Resource (`Core/Domain/{Domain}/Resources`) |
| Business rules, mutations, workflows | Never here → see the labrodev-action skill |

## Rules

### ViewModels — Musts

- Live in `App/Layer/{Layer}/{Domain}/ViewModels`; class is `final`, extends `Spatie\ViewModels\ViewModel` directly.
- Named `{Model}{Context}ViewModel` (`BookingIndexViewModel`, `BookingShowViewModel`). Full naming rules → see the labrodev-naming skill.
- **Always override `toArray()`** returning `array<string, mixed>`. The controller passes `$viewModel->toArray()` to `Inertia::render(...)`; Spatie's magic view-data features are not used.
- All input is **constructor-injected** (`private readonly` properties): models, paginators, DTOs, query results. The controller/IndexQuery fetches the data → see the labrodev-query skill.
- Resources are **always materialized with `->resolve()`** inside the ViewModel — never returned as `JsonResource` instances to Inertia.
- Paginated collections use the canonical **pagination envelope** (see index template below): `data`, `current_page`, `last_page`, `per_page`, `total`, `from`, `to`, `links`.
- Select/filter option props (`status_options`, `options`, ...) originate in the ViewModel — the frontend never hardcodes option values. Enum option emission (value + label via EnumMapper) → see the labrodev-enum skill.
- Show/edit ViewModels load required relations (`$this->model->load([...])`) inside `toArray()` before resolving the Resource.

### ViewModels — Must-nots

- No business logic, no mutations, no Action/Service/Job calls — presentation shaping and mapping only.
- No raw models, `$model->toArray()`, or unfiltered collections in the returned array — everything goes through a Resource.
- No UI translation payloads (`translations` prop or equivalent) for Inertia pages — copy is handled by frontend i18n → see the labrodev-inertia-react skill.
- Domain code (Actions, Services, Queries) must never construct or return ViewModels or Resources — they are instantiated at the delivery boundary (controller/ViewModel).

### Resources — Musts

- Live in `Core/Domain/{Domain}/Resources`; class is `final`, extends `Illuminate\Http\Resources\Json\JsonResource`, with a `JsonResource<Model>` class docblock.
- Named `{Model}{Context}Resource` (`BookingIndexResource`, `BookingShowResource`). One model MAY have multiple Resources per context: public API, internal API, dashboard UI, Inertia view data.
- `toArray()` is an **explicit allowlist**: only fields the specific page/consumer actually renders. Every field must have a corresponding UI element.
- **`uuid` is the public identifier** — expose `uuid`, never the internal integer `id` (unless the page explicitly displays it, which is almost never).
- Include the **`fetchModel()` typed guard**: verify `$this->resource instanceof Model`, throw `ObjectMissed::make(Model::class)` otherwise, and return the typed model. All field access goes through the guarded variable.
- Field grammar: snake_case keys; datetimes as `->toDateTimeString()`, dates as `->toDateString()`; enum fields as `value` + `*_label` pairs — full enum contract → see the labrodev-enum skill.
- May compute derived presentation values (formatting, totals, concatenated names).

### Resources — Must-nots

- Must not mutate state.
- Must not trigger workflows (Actions, Jobs).
- Must not spread `$model->toArray()` / `attributesToArray()` into the output — that exposes every attribute and loaded relation.
- Must not expose: internal integer ids, password hashes, tokens/secrets, internal system fields (`created_by`, internal flags) unless explicitly displayed, pivot/debug metadata.

## Template — Index ViewModel

Naming pattern: `{Model}IndexViewModel` in `App\Layer\{Layer}\{Domain}\ViewModels`. Worked example: Booking domain, Dashboard layer.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\ViewModels;

use Core\Domain\Booking\Resources\BookingIndexResource;
use Illuminate\Pagination\LengthAwarePaginator;
use Spatie\ViewModels\ViewModel;

final class BookingIndexViewModel extends ViewModel
{
    public function __construct(
        private readonly LengthAwarePaginator $items,
    ) {}

    /**
     * @return array<string, mixed>
     */
    public function toArray(): array
    {
        $resourceCollection = BookingIndexResource::collection(
            $this->items->getCollection()
        )->resolve();

        return [
            'bookings' => [
                'data' => $resourceCollection,
                'current_page' => $this->items->currentPage(),
                'last_page' => $this->items->lastPage(),
                'per_page' => $this->items->perPage(),
                'total' => $this->items->total(),
                'from' => $this->items->firstItem(),
                'to' => $this->items->lastItem(),
                'links' => $this->items->linkCollection()->toArray(),
            ],

            // Optional extra props: filter/select options, statuses, etc.
            // Enum select/filter options → see the labrodev-enum skill.
        ];
    }
}
```

The controller constructs it with named arguments (`new BookingIndexViewModel(items: $bookings)`) and renders `Inertia::render('bookings/index', $viewModel->toArray())` → see the labrodev-controller skill.

## Template — Show/Edit ViewModel

Naming pattern: `{Model}ShowViewModel` (or `{Model}EditViewModel` for form pages) in the same namespace.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\ViewModels;

use Core\Domain\Booking\Models\Booking;
use Core\Domain\Booking\Resources\BookingShowResource;
use Spatie\ViewModels\ViewModel;

final class BookingShowViewModel extends ViewModel
{
    public function __construct(
        private readonly Booking $booking,
    ) {}

    /**
     * @return array<string, mixed>
     */
    public function toArray(): array
    {
        $this->booking->load([
            // 'customer',
            // 'items.service',
        ]);

        return [
            'booking' => (new BookingShowResource($this->booking))->resolve(),

            // Optional: form metadata (select options, permissions, statuses).
            // 'status_options' => ...,
        ];
    }
}
```

## Template — Resource

Naming pattern: `{Model}{Context}Resource` in `Core\Domain\{Domain}\Resources`.

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Resources;

use Core\Domain\Booking\Models\Booking;
use Core\Shared\Exceptions\Models\ObjectMissed;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

/**
 * JsonResource<Booking>
 */
final class BookingIndexResource extends JsonResource
{
    /**
     * @return array<string, mixed>
     * @throws ObjectMissed
     */
    public function toArray(Request $request): array
    {
        $booking = $this->fetchModel();

        return [
            'uuid' => $booking->uuid,
            'reference' => $booking->reference,
            'status' => $booking->status->value,
            'status_label' => $booking->status->label(),
            'starts_at' => $booking->starts_at?->toDateTimeString(),
            'created_at' => $booking->created_at?->toDateTimeString(),
        ];
    }

    /**
     * @throws ObjectMissed
     */
    private function fetchModel(): Booking
    {
        if (! $this->resource instanceof Booking) {
            throw ObjectMissed::make(Booking::class);
        }

        return $this->resource;
    }
}
```

## Props security — everything serializes into the DOM

Inertia serializes **all** page props into the `data-page` attribute of the HTML root element. Every prop is visible to anyone inspecting the page source. Therefore:

- Never pass raw models, full DB records, or unfiltered collections — every value passes through a Resource or a manually constructed array.
- Audit eager-loaded relations: if the ViewModel loads `$model->load(['relation'])`, the nested relation must also go through its own allowlisting Resource. Nested relations are the most common leak source.
- When adding a field to a Resource ask: "Does the frontend render this?" If no — remove it.
- During development, inspect `data-page` in the browser. Anything visible there is public; if it should not be, fix the Resource or ViewModel.
- Defense in depth on the model side (`$visible`) → see the labrodev-model skill.

## Edge cases

- **Blade / classic MVC ViewModels:** same base class and `toArray()` override, but custom helper methods (`title()`, `options()`, a `template(): string` returning the Blade view path) are allowed and expected. Translation enumeration rules for Inertia do not apply to Blade ViewModels.
- **Create vs edit forms:** a form page ViewModel may accept a nullable model (`?Booking $booking = null`) and emit `null` for the model prop in create mode; the frontend derives the mode from the presence of `uuid`.
- **Wrong resource type at runtime:** the `fetchModel()` guard turns a mis-wired collection (e.g. passing an array or a different model) into an explicit `ObjectMissed` exception instead of a silent property-access failure.
- **Multiple contexts:** never grow one Resource to serve every page. If the show page needs more fields than the index table, create `BookingShowResource` alongside `BookingIndexResource` — allowlists stay minimal per context.
- **JsonControllers** return Resources/collections directly instead of Inertia props; the Resource rules here apply unchanged — controller shape → see the labrodev-controller skill.
- **Exports** may consume IndexQueries/ViewModels but never mutate state — the same "no raw models out" rule applies.

## Review checklist

1. Is the ViewModel `final`, extending `Spatie\ViewModels\ViewModel`, with `toArray()` overridden?
2. Are all inputs constructor-injected as `private readonly`, with no DB queries in the ViewModel (relation `load()` in show/edit ViewModels excepted)?
3. Is every model materialized through a Resource with `->resolve()` — no `JsonResource` instances and no raw models in props?
4. Does the paginated prop use the canonical envelope (`data`, `current_page`, `last_page`, `per_page`, `total`, `from`, `to`, `links`)?
5. Is the Resource `final`, in `Core/Domain/{Domain}/Resources`, with the `JsonResource<Model>` docblock and the `fetchModel()` guard throwing `ObjectMissed`?
6. Does the Resource expose `uuid` (never internal integer `id`) and only fields the page actually renders?
7. Do datetime fields use `->toDateTimeString()` / `->toDateString()`, and enum fields emit `value` + `*_label`?
8. Are nested/eager-loaded relations also passed through allowlisting Resources?
9. Is the output free of secrets, tokens, password hashes, internal flags, and pivot/debug metadata?
10. Is there zero business logic, mutation, or Action/Job dispatch in either class?
