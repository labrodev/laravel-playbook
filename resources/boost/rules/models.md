---
paths:
  - "**/Core/Domain/**/Models/**"
  - "**/Observers/**"
  - "**/Collections/**"
---

# Models, Observers & Collections — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular; the class body contains no comments — typed PHPStan annotations only.
- Model extends `BaseModel` and contains ONLY: wiring class attributes, `$visible`, `casts()`, relations, and persistence-level Eloquent `Attribute` accessors — no business, querying, or rendering logic, no helper methods, no scopes.
- The mandatory trio exists: `{Model}Policy`, `{Model}Observer`, and `{Model}Collection` for every model.
- `#[Table]`, `#[ObservedBy]`, `#[CollectedBy]`, `#[UsePolicy]` (and `#[UseFactory]` once a factory exists) are declared as class attributes — no `protected $table`, no `boot()` wiring, no `newCollection()` override.
- Model carries no `@property` lists and no comments — only the `/** @mixin IdeHelper{Model} */` line, with column metadata delegated to the regenerated ide-helper mixin.
- Casts are declared in the `casts()` method (never `$casts`), covering every enum, date, float, int, and JSON field.
- Every relation method declares the correct native Relation return type plus the `@return` generics docblock.
- No `App\` imports — user relations point at the Core mirror `Core/Domain/User/Models/User`, never `App\Models\User`.
- `$visible` is explicitly defined; no `$fillable` anywhere.
- A `{Model}Collection` exists with `@extends Collection<int,{Model}>` and read-only helpers returning `static` with `->values()`.
- The `{Model}Observer` is `final readonly` and limited to logging (default), cache invalidation, or async events — nothing the main flow depends on; no notifications sent from Observers (chain → rules/actions.md).
- UUID generation is absent from the model, `boot()`, and traits (assigned in the create Action); `getRouteKeyName()` is not overridden.

Full anatomy and templates → labrodev-model skill.
