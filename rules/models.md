---
paths:
  - "**/Core/Domain/**/Models/**"
  - "**/Observers/**"
  - "**/Collections/**"
---

# Models, Observers & Collections — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular.
- Model extends `BaseModel` and contains only state, `casts()`, and relations — no workflows, queries, or scopes.
- `#[Table]`, `#[ObservedBy]`, `#[CollectedBy]`, `#[UsePolicy]` (and `#[UseFactory]` once a factory exists) are declared as class attributes — no `protected $table`, no `boot()` wiring, no `newCollection()` override.
- Model carries no `@property` lists and no comments — only the `/** @mixin IdeHelper{Model} */` line, with column metadata delegated to the regenerated ide-helper mixin.
- Casts are declared in the `casts()` method (never `$casts`), covering every enum, date, float, int, and JSON field.
- Every relation method declares the correct native Relation return type plus the `@return` generics docblock.
- `$visible` is explicitly defined; no `$fillable` anywhere.
- A `{Model}Collection` exists with `@extends Collection<int,{Model}>` and read-only helpers returning `static` with `->values()`.
- Observer (if any) is `final readonly` and limited to logging, cache invalidation, or async events — nothing the main flow depends on.
- UUID generation is absent from the model, `boot()`, and traits (assigned in the create Action); `getRouteKeyName()` is not overridden.

Full anatomy and templates → labrodev-model skill.
