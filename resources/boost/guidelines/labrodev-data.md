# Labrodev Data Classes

Always-on law. Spatie Data classes are the **only** input-mapping and validation layer — there are no Request/FormRequest classes in this architecture, ever. A Data class is strictly a **mapper and holder of request inputs**: it carries what the client submitted, validated and typed — nothing else. Data classes live in `Core\Domain\{Domain}\Data`; casters in `Core\Domain\{Domain}\Casts`. Anatomy and templates → the `labrodev-data` skill; per-file checks → `rules/data.md`.

## Musts

- **Every write operation (store, update) takes exactly one Data class**; it defines the allowed input fields, validation rules, and casts.
- **One shared `{Model}Data` is the default** — a single class serves both the create and the update Action. Split into `{Model}CreateData` / `{Model}UpdateData` ONLY when the two operations genuinely accept different fields, never pre-emptively.
- Data classes extend `Spatie\LaravelData\Data`, are `final`, and live in `Core\Domain\{Domain}\Data`.
- **All validation rules live in static `rules()`** inside the Data class. Validation always targets the **raw input keys**; casting happens after validation.
- Provide static `attributes()` with `trans()` labels; its keys must match `rules()` keys exactly.
- All properties are `snake_case` and match incoming input keys exactly (`$starts_at`, `$guest_count`).
- **Relations follow the four-part UUID contract**: entity-name key, typed model property, `#[WithCast(XUuidCaster::class)]`, and an `exists:<table>,uuid` rule on the same key.
- **Input normalization happens in `prepareForPipeline()`** (trim, lowercase, numeric coercion) — it runs before validation and casting.
- **Casters resolve models exclusively through the Domain Query class** and return `Uncastable::create()` (or an empty collection) for bad input — never throw.
- Controllers receive Data via method injection — type-hint the Data class in `__invoke()`; Spatie Data resolves it from the request automatically.
- Domain code (Actions) assumes it receives valid, already-mapped Data objects.

## Must-nots

- Never create a FormRequest, never call `$request->validate()`, never validate in a controller or Action.
- Never mark a Data class `readonly` — `Spatie\LaravelData\Data` is not readonly, so a readonly subclass is a fatal error in PHP 8.2+.
- Never create Data objects for the read side: index, filtering, sorting, pagination, and exports take no Data, no Request class, and no validation — read-side input is tolerant and handled defensively in queries (→ labrodev-query).
- Never expose internal database IDs: no `$service_id`, no `$service_uuid`, no `_id`/`_uuid` suffixed field that references a related model.
- Never construct Data manually in controllers: no `Illuminate\Http\Request` injection, no `MyData::from($request->all())`, no `$request->input()` plucking.
- Never put computed/derived fields in a Data class (calculated price, status set by the Action, timestamps). If the client never submits it, it does not belong here.
- Never put business logic, authorization decisions, domain-state mutation, or workflows in a Data class — business decisions belong to Core logic (→ labrodev-action).
- Never use a Data class as anything but the request-input carrier: not as an internal DTO between Core classes (boundary DTOs → labrodev-infrastructure, flow state → Payloads, labrodev-pipeline), not as an Action's return value, not as an output/response shape (→ labrodev-viewmodel-resource).
- Never express validation failures as domain exceptions — validation is a delivery-layer concern and stops execution before Core logic runs.
- Enum keys follow the enum contract: typed enum property + `Rule::enum(...)`, never `in:` lists (→ labrodev-enum).
