---
paths:
  - "**/Core/Domain/**/Queries/**"
  - "app/Layer/**/IndexQueries/**"
---

# Queries & IndexQueries — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular; the class body contains no comments — typed PHPStan annotations only.
- Exactly one `{Model}Query` per model, in `Core/Domain/{Domain}/Queries/`, with the four baseline methods (`all()`, `byId()`, `byUuid()`, `byUuids()`).
- Every composable Query method returns `Builder` with a `@return Builder<Model>` docblock (and `@param array<int, string>` where applicable).
- Scalar-returning methods are limited to explicitly named terminal reads with thin terminal bodies (`exists()`, `count()`, `value()`).
- Every `Model::query()` / `DB::table()` / `DB::select()` call sits inside a Query class or an IndexQuery constructor — none anywhere else (Actions, Services, ViewModels, Controllers, Resources, Rules, Observers, Jobs); callers start from a Query class and may chain further conditions on the returned `Builder`.
- IndexQuery filters are standard `AllowedFilter::exact` / `AllowedFilter::partial` over standard joins/left joins — custom callbacks only where no standard filter can express the condition.
- The IndexQuery lives in `App/Layer/{Layer}/{Domain}/IndexQueries/`, is `final`, carries `@extends QueryBuilder<Model>`, and wires base query + `defaultSort` + `allowedFilters` + `allowedSorts` in the constructor.
- Client-facing filter keys identify records by uuid (own and related), never by internal id.
- Filter/sort columns are table-qualified, mandatory once joins exist.
- Queries and IndexQueries are free of writes, state mutation, Action calls, and business rules.
- New Query methods are justified by an actual business use case, not added speculatively.

Full anatomy and templates → labrodev-query skill.
