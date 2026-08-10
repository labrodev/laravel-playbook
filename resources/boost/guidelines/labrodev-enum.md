# Labrodev Enums

Always-on law. An enum represents a constrained domain value — a status, type, or mode with a finite allowed set and matching logic. When an enum crosses a boundary, the full enum contract applies: the enum class itself, Data validation, the model cast, and backend-resolved labels on the frontend. Anatomy and templates → the `labrodev-enum` skill; per-file checks → `rules/enums.md`.

## Musts

- **Enums live in `Core\Domain\{Domain}\Enums`** as **backed** enums (`int` or `string`); the backing type is chosen for domain meaning and persistence needs, not storage convenience.
- **Model-prefixed names**: an enum tied to a model starts with the model name — `BookingStatus`, `ProductType`, `OrderState`. Generic names (`Status`, `Type`, `State`) are forbidden.
- **The presentation helper is always named `label(): string`** and wraps `trans()` — EnumMapper and Resources rely on that exact name.
- **`label()` uses an exhaustive `match ($this)`** — no `default` arm; a new case must fail loudly until it gets a label.
- **Enum helper methods are pure** (`label()`, optional `color()`, `icon()`, `shortLabel()`): no side effects, no Actions/Services/Jobs, no external APIs, no business workflows.
- **Static domain lookups** are named constructors wrapping `tryFrom()` — they return `?self` and never throw.
- **Data classes type-hint the enum directly** (Spatie casts the raw scalar implicitly — no `#[WithCast]` in the common case) and always validate the raw input key with `Rule::enum(BookingStatus::class)` (→ labrodev-data).
- **Every enum backing a model field is cast in that model's `casts()` method**: `'status' => BookingStatus::class`.
- **Resources emit value + label pairs**: `'status' => $this->status->value`, `'status_label' => $this->status->label()` — labels are resolved on the backend.
- **Select/filter option maps are built in ViewModels** via `EnumMapper::keyValues(BookingStatus::cases(), 'label')` (`labrodev/php-enum-mapper`) — never hardcoded in the frontend.
- **The frontend renders `*_label` for display** and uses the raw `value` for logic/filters; TypeScript literal unions mirror the enum values.
- **Unknown enum values get an explicit frontend fallback** (the raw value) — never silently fail or render nothing.

## Must-nots

- No pure (unbacked) enums for domain values that cross a boundary — persistence and validation need the scalar.
- No `in:` lists or raw value arrays for enum validation — always `Rule::enum(EnumClass::class)`.
- No enum cast in a `$casts` property — enum fields are cast in the `casts()` method (project-wide `$casts` prohibition → labrodev-model).
- No enum label resolution in the frontend via `t()` — labels arrive pre-translated (`*_label` fields, EnumMapper option maps); the one deliberate exception to frontend-only copy (→ labrodev-inertia-react).
- No emitting only the raw value and letting React map value → text — that duplicates the label source of truth.
- No workflows, state transitions, or cross-entity logic in an enum — whether a status may change belongs to domain Rules and Actions (→ labrodev-action).
- No calls to Actions, Services, Jobs, queries, or external APIs from enum methods.
- No presentation helper named anything other than `label()` — `getLabel()`, `title()`, `name()` break the EnumMapper/Resource convention.
