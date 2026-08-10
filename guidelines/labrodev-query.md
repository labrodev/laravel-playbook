# Labrodev Queries — Core Query & Layer IndexQuery

Always-on law. All read-side fetching routes through two families: `{Model}Query` in `Core/Domain/{Domain}/Queries/` — the single source of truth for querying that model — and `{Model}IndexQuery` in `App/Layer/{Layer}/{Domain}/IndexQueries/` — one interface's listing needs (user-driven filters, sorts, pagination). A Layer module has no generic `Queries/` folder and Core has no IndexQueries — user-facing listing concerns are delivery concerns. Anatomy and templates → the `labrodev-query` skill; per-file checks → `rules/queries.md`.

## Musts

- **Exactly one Query class per Model**, named `{Model}Query`, in `Core/Domain/{Domain}/Queries/` — all read-side composition for that model routes through it.
- Every Query method uses `{Model}::query()` as its entry point.
- Query methods return `Builder` and carry a PHPDoc generic: `@return Builder<Booking>`. PHPStan must see the model type.
- Ship the baseline methods on every Query: `all()`, `byId(int $id)`, `byUuid(string $uuid)`, `byUuids(array $uuids)` (with `@param array<int, string> $uuids`).
- Add further methods **only when business logic demands them**; keep them composable so callers chain: `$bookingQuery->byUuid($uuid)->firstOrFail()`.
- Resolution (`->get()`, `->first()`, `->paginate()`, `->exists()`) happens at the caller's edge, not inside composable Query methods.
- IndexQueries are named `{Model}IndexQuery` and extend `Spatie\QueryBuilder\QueryBuilder` with `@extends QueryBuilder<Booking>`.
- The IndexQuery constructor takes `Request $request`, builds the base query (eager loads, joins, selects), calls `parent::__construct($query, $request)`, then declares `defaultSort`, `allowedFilters`, `allowedSorts`.
- Qualify filter/sort columns with the table name (`bookings.created_at`) — mandatory once joins exist.
- Client-facing filter keys identify records by **uuid**, never by internal auto-increment id — internal ids are implementation details, uuids are the public contract.
- Obtain Query classes via constructor or method injection — they are stateless and container-resolvable.

## Must-nots

- Never write inline `Model::query()` (or `Model::where(...)`, `DB::table(...)`) in Actions, Services, ViewModels, Controllers, or Resources — `Model::query()` may appear only inside the model's Query class and an IndexQuery constructor.
- Query methods must not mutate state, call Actions, or perform writes or side effects. Read side only.
- Composable-looking methods (`all()`, `active()`, `forCustomer()`) must not return `int`, `Collection`, arrays, or other resolved results — `Builder` only (terminal-read exception → labrodev-query skill).
- IndexQueries must not mutate domain state and must not contain business rules — listing concerns only (filters, sorts, columns, joins, eager loads).
- Never put an IndexQuery in Core, and never duplicate one IndexQuery's logic into another Layer — each Layer defines its own.
- File header contract (`declare(strict_types=1)`, `final`) (→ labrodev-core); naming beyond the patterns shown here (→ labrodev-naming).
