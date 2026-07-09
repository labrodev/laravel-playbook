---
name: labrodev-query
description: "Use when creating, reviewing, or extending read-side query classes in a Labrodev Laravel project: Core Domain Query classes (e.g. BookingQuery) and Layer IndexQuery classes (e.g. BookingIndexQuery), or when deciding where any data-fetching code (listing, filtering, sorting, existence checks) belongs."
license: MIT
metadata:
  author: labrodev
---

# Read side: Core Queries and Layer IndexQueries

Part of the Labrodev playbook skill set — assumes labrodev-core and labrodev-naming are installed. If absent, minimum global rules: `declare(strict_types=1)`, final classes, App/Layer depends on Core (never the reverse), named-argument invocation.

Two read-side class families exist, with different homes and jobs:

| Class | Location | Job |
|---|---|---|
| `{Model}Query` | `Core/Domain/{Domain}/Queries/` | Single source of truth for querying that model. Composable `Builder` methods for business reads. |
| `{Model}IndexQuery` | `App/Layer/{Layer}/{Domain}/IndexQueries/` | Spatie QueryBuilder subclass for one interface's listing needs: user-driven filters, sorts, pagination, table columns. |

A Layer module has **no** generic `Queries/` folder — only `IndexQueries/`. Core has **no** IndexQueries — user-facing listing concerns are delivery concerns.

## Rules

### Musts

- Exactly **one Query class per Model**, named `{Model}Query`, in `Core/Domain/{Domain}/Queries/`. All read-side composition for that model routes through it.
- Every Query method uses `{Model}::query()` as its entry point.
- Query methods return `Builder` and carry a PHPDoc generic: `@return Builder<Booking>`. PHPStan must see the model type.
- Ship the baseline methods on every Query: `all()`, `byId(int $id)`, `byUuid(string $uuid)`, `byUuids(array $uuids)` (with `@param array<int, string> $uuids`).
- Add further methods **only when business logic demands them**; keep them composable so callers chain: `$bookingQuery->byUuid($uuid)->firstOrFail()`.
- Resolution (`->get()`, `->first()`, `->paginate()`, `->exists()`) happens at the caller's edge, not inside composable Query methods.
- IndexQueries extend `Spatie\QueryBuilder\QueryBuilder` with `@extends QueryBuilder<Booking>` and are named `{Model}IndexQuery`.
- IndexQuery constructor takes `Request $request`, builds the base query (eager loads, joins, selects), calls `parent::__construct($query, $request)`, then declares `defaultSort`, `allowedFilters`, `allowedSorts`.
- Qualify columns with the table name (`bookings.created_at`) in filters/sorts, mandatory once joins exist.
- Filter keys exposed to the client identify records by **uuid**, never by internal auto-increment id. Internal ids are implementation details; UUIDs are the public contract.
- Obtain Query classes via constructor or method injection (they are stateless and container-resolvable).

### Must-nots

- Never write inline `Model::query()` (or `Model::where(...)`, `DB::table(...)`) in Actions, Services, ViewModels, Controllers, or Resources. `Model::query()` may appear only inside the model's Query class and inside an IndexQuery constructor.
- Query methods must not mutate state, call Actions, or perform writes or side effects. Read side only.
- Composable-looking methods (`all()`, `active()`, `forCustomer()`) must not return `int`, `Collection`, arrays, or other resolved results — `Builder` only. (See the terminal-read exception below.)
- IndexQueries must not mutate domain state and must not contain business rules — listing concerns only (filters, sorts, columns, joins, eager loads).
- Do not put an IndexQuery in Core, and do not duplicate one IndexQuery's logic into another Layer — each Layer defines its own.
- File header contract (`declare(strict_types=1)`, `final`) → see the labrodev-core skill.
- Class/method/variable naming rules beyond the patterns shown here → see the labrodev-naming skill.

## Template: Core Query class

Naming pattern: `{Model}Query` in `Core\Domain\{Domain}\Queries`. Worked example — Booking domain:

```php
<?php

declare(strict_types=1);

namespace Core\Domain\Booking\Queries;

use Core\Domain\Booking\Models\Booking;
use Core\Domain\Customer\Models\Customer;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Support\Carbon;

final class BookingQuery
{
    /**
     * @return Builder<Booking>
     */
    public function all(): Builder
    {
        return Booking::query();
    }

    /**
     * @return Builder<Booking>
     */
    public function byId(int $id): Builder
    {
        return Booking::query()->where('id', '=', $id);
    }

    /**
     * @return Builder<Booking>
     */
    public function byUuid(string $uuid): Builder
    {
        return Booking::query()->where('uuid', '=', $uuid);
    }

    /**
     * @param  array<int, string>  $uuids
     * @return Builder<Booking>
     */
    public function byUuids(array $uuids): Builder
    {
        return Booking::query()->whereIn('uuid', $uuids);
    }

    // Business-driven methods — add only when a use case needs them.

    /**
     * @return Builder<Booking>
     */
    public function upcomingForCustomer(Customer $customer): Builder
    {
        return Booking::query()
            ->where('customer_id', '=', $customer->id)
            ->where('starts_at', '>=', Carbon::now());
    }

    // Terminal-read exception: name states the scalar outcome,
    // body is a thin terminal on the builder.
    public function existsForCustomerOnDate(Customer $customer, Carbon $date): bool
    {
        return Booking::query()
            ->where('customer_id', '=', $customer->id)
            ->whereDate('starts_at', '=', $date)
            ->exists();
    }
}
```

## Template: Layer IndexQuery

Naming pattern: `{Model}IndexQuery` in `App\Layer\{Layer}\{Domain}\IndexQueries`. Worked example — Dashboard layer, Booking domain (table `bookings`):

```php
<?php

declare(strict_types=1);

namespace App\Layer\Dashboard\Booking\IndexQueries;

use Core\Domain\Booking\Models\Booking;
use Illuminate\Http\Request;
use Spatie\QueryBuilder\AllowedFilter;
use Spatie\QueryBuilder\AllowedSort;
use Spatie\QueryBuilder\QueryBuilder;

// Optional (when labrodev/filters is installed):
// use Labrodev\Filters\QueryBuilder\DateRangeFilter;
// use Labrodev\Filters\QueryBuilder\IsNotNullFilter;
// use Labrodev\Filters\QueryBuilder\WhereInFilter;

/**
 * @extends QueryBuilder<Booking>
 */
final class BookingIndexQuery extends QueryBuilder
{
    public function __construct(Request $request)
    {
        $query = Booking::query()
            ->with([
                'customer',
            ])
            // Optional joins for searching/sorting by related fields:
            // ->leftJoin('customers', 'customers.id', '=', 'bookings.customer_id')
            // ->select(['bookings.*', 'customers.name as customer_name'])
        ;

        parent::__construct($query, $request);

        // Default sort is internal — prefer explicit table/alias once joins exist.
        $this->defaultSort('-bookings.id');

        $this->allowedFilters([
            // Public record identity is always uuid, never internal id:
            AllowedFilter::exact('uuid', 'bookings.uuid'),

            // Partials (like):
            AllowedFilter::partial('reference', 'bookings.reference'),

            // Related records are filtered by their uuid as well:
            AllowedFilter::callback('customer', static function ($query, $value): void {
                $query->whereHas('customer', static function ($query) use ($value): void {
                    $query->where('customers.uuid', '=', $value);
                });
            }),

            // Date range (labrodev/filters, when installed):
            // AllowedFilter::custom('created_at', new DateRangeFilter, 'bookings.created_at'),
        ]);

        $this->allowedSorts([
            AllowedSort::field('created_at', 'bookings.created_at'),
            AllowedSort::field('updated_at', 'bookings.updated_at'),
            AllowedSort::field('starts_at', 'bookings.starts_at'),
            // Sort by joined alias:
            // AllowedSort::field('customer', 'customer_name'),
        ]);
    }
}
```

The controller injects the IndexQuery and resolves it (`$bookingIndexQuery->paginate(...)->withQueryString()`) → see the labrodev-controller skill for the controller anatomy and the labrodev-viewmodel-resource skill for shaping the result.

## IndexQuery vs plain Query — which one?

- **IndexQuery**: an index/listing page (or export) where the *client* drives filters, sorts, or pagination via request parameters. One per model per Layer that lists it.
- **Core Query**: every other read — fetching a record for a mutation, existence/count checks inside Actions and Rules, fixed lists (dropdown options, related records for a show page), any read whose shape is decided by *business logic* rather than by request parameters.
- An index page with no user-driven filtering/sorting can still use the Core Query; introduce the IndexQuery when the first filter or sort parameter appears.
- Exports may reuse the same IndexQuery as the index page — never fork a second listing query for the same table.

## Edge cases

- **Terminal reads**: a Query method may return `bool`/`int` only when the name states the outcome (`existsForCustomerOnDate(): bool`, `countForDate(): int`) and the body is a thin terminal (`exists()`, `count()`, `value()`). Never from methods that read like composable scopes.
- **Cross-model reads**: a method belongs to the Query of the model it *returns*. `BookingQuery::upcomingForCustomer()` is correct; do not put booking reads on a `CustomerQuery`.
- **Eager loading**: `->with(...)` inside Query/IndexQuery methods is fine and encouraged to prevent N+1; relation definitions and their PHPStan generics docblocks → see the labrodev-model skill.
- **Enum filtering**: filter/where on the enum's backed value; the enum contract itself → see the labrodev-enum skill.
- **Route-bound models**: a controller receiving a model via `{booking:uuid}` binding does not need a Query call for that record — Queries cover reads *beyond* the bound instance. Binding rules → see the labrodev-controller skill.
- **Authorization**: never inside Query classes → see the labrodev-authorization skill.
- **Testing Queries/IndexQueries** → see the labrodev-testing skill.

## Review checklist

1. Is there exactly one `{Model}Query` per model, in `Core/Domain/{Domain}/Queries/`, with the four baseline methods?
2. Does every composable Query method return `Builder` with a `@return Builder<Model>` docblock (and `@param array<int, string>` where applicable)?
3. Are scalar-returning methods limited to explicitly named terminal reads with thin terminal bodies?
4. Is every `Model::query()` call located inside a Query class or an IndexQuery constructor — none inline in Actions, Services, ViewModels, Controllers, or Resources?
5. Is the IndexQuery in `App/Layer/{Layer}/{Domain}/IndexQueries/`, `final`, with `@extends QueryBuilder<Model>`, wiring base query + `defaultSort` + `allowedFilters` + `allowedSorts` in the constructor?
6. Do all client-facing filter keys identify records by uuid (own and related), never by internal id?
7. Are filter/sort columns table-qualified, especially where joins exist?
8. Are Query and IndexQuery free of writes, state mutation, Action calls, and business rules?
9. Are new Query methods justified by an actual business use case rather than added speculatively?
