---
paths:
  - "**/Core/Domain/**/Data/**"
  - "**/Casts/**"
---

# Data classes & Casters — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular.
- Every write endpoint is backed by exactly one Data class — no FormRequest, `$request->validate()`, or controller-side validation anywhere.
- The Data class is `final` but NOT `readonly`; all properties are `snake_case` matching the raw input keys.
- Every relation follows the four-part UUID contract (entity-name key, typed model property, `#[WithCast(XUuidCaster::class)]`, `exists:<table>,uuid` rule on the same key) — no `_id`/`_uuid` fields.
- `rules()` validates raw input keys only; `attributes()` keys match `rules()` keys exactly with `trans()` labels.
- Input normalization lives in `prepareForPipeline()` — not in controllers, casters, or Actions.
- `prepareForPipeline()` normalizes the format of the declared input keys (trim, case, shape) — it never reads alternative key spellings or invents values for required fields (→ labrodev-core contract commitment).
- Casters resolve models via the Domain Query class, pass through model/collection instances unchanged, and return `Uncastable::create()` (or an empty collection) instead of throwing.
- Enum keys follow the enum contract (→ labrodev-enum).
- Nested Data collections are declared with `#[DataCollectionOf(...)]` and validated with wildcard rules in the parent.
- The Data class is free of computed/derived fields, business logic, authorization, and state mutation.
- The read side is completely free of Data classes and validation.

Full anatomy and templates → labrodev-data skill.
