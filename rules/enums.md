---
paths:
  - "**/Core/Domain/**/Enums/**"
---

# Enums — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; names are singular.
- The enum is a backed scalar enum (`int` or `string`) in `Core\Domain\{Domain}\Enums`.
- The name is model-prefixed (`BookingStatus`), not generic (`Status`, `Type`, `State`).
- `label(): string` exists, wraps `trans()`, and uses an exhaustive `match` with no `default` arm.
- All enum methods are pure — no Actions, Services, Jobs, queries, external APIs, or workflows.
- Every Data class touching the enum uses a typed enum property AND `Rule::enum(EnumClass::class)` — no `in:` lists.
- Every model field backed by the enum is cast in the `casts()` method — never a `$casts` property.
- Resources emit both `'field' => ->value` and `'field_label' => ->label()` (null-safe for nullable fields).
- Select/filter options come from `EnumMapper::keyValues(Enum::cases(), 'label')` in a ViewModel — never hardcoded in React.
- The frontend renders `*_label` for display, uses the raw value for logic, and shows an explicit raw-value fallback for unknown values.
- Static lookups are named constructors wrapping `tryFrom()`, returning `?self` — never throwing.

Full anatomy and templates → labrodev-enum skill.
