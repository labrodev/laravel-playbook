---
name: labrodev-viewmodel-resource
description: "Use when creating, reviewing, or naming ViewModels (App/Layer/*/ViewModels, e.g. BookingIndexViewModel, BookingShowViewModel) or Resources (App/Layer/*/Resources, e.g. BookingIndexResource) — i.e. whenever data is shaped for an Inertia page, Blade template, or JSON payload, or when deciding which fields a model may expose to the browser."
license: MIT
metadata:
  author: labrodev
---

# ViewModels & Resources — the presentation boundary

Part of the Labrodev playbook. **The law for this component lives in the always-on `labrodev-viewmodel-resource` guideline** (musts, must-nots); the per-file checklist is `rules/viewmodels-resources.md`. This skill holds the craft: anatomy, canonical templates, and edge cases.

Together these two classes form the security boundary between the database and the browser:

- The **ViewModel** decides *which* data reaches the page (which models, which relations, which options).
- The **Resource** decides *what fields* of that data are exposed (an explicit allowlist).

Neither may be bypassed. Raw Eloquent models never reach Inertia or JSON output.

## Responsibility split (controller vs ViewModel vs Resource)

| Concern | Owner |
|---|---|
| Authorize, run IndexQuery, paginate, construct ViewModel, `Inertia::render(...)` | Controller → see the labrodev-controller skill |
| Collect/shape all props for the page: pagination envelope, resolved Resources, select options | ViewModel (`App/Layer/{Layer}/{Domain}/ViewModels`) |
| Allowlist mapping of a single model to public fields | Resource (`App/Layer/{Layer}/{Domain}/Resources`) |
| Business rules, mutations, workflows | Never here → see the labrodev-action skill |

## Template — Index ViewModel

Naming pattern: `{Model}IndexViewModel` in `App\Layer\{Layer}\{Domain}\ViewModels`. Worked example: Booking domain, Dashboard layer.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\ViewModels;

use App\Layer\Dashboard\Booking\Resources\BookingIndexResource;
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

use App\Layer\Dashboard\Booking\Resources\BookingShowResource;
use Core\Domain\Booking\Models\Booking;
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

Naming pattern: `{Model}{Context}Resource` in `App\Layer\{Layer}\{Domain}\Resources`. Worked example: Booking domain, Dashboard layer.

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\Resources;

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
- **Per-layer ownership:** each delivery surface owns its Resources — Dashboard and Api each keep their own `BookingResource`, never shared through Core.
- **JsonControllers** return Resources/collections directly instead of Inertia props; the Resource rules here apply unchanged — controller shape → see the labrodev-controller skill.
- **Exports** may consume IndexQueries/ViewModels but never mutate state — the same "no raw models out" rule applies.
