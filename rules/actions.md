---
paths:
  - "**/Core/Domain/**/Actions/**"
  - "**/Core/Domain/**/Services/**"
  - "**/Core/Domain/**/Rules/**"
---

# Actions & Services — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular.
- Actions live in `Core/Domain/{Domain}/Actions/`, are `final readonly`, and expose exactly one public `__invoke()`.
- All business-condition guards run before any mutation and delegate to static `{Model}Rule` methods.
- The correct failure mode is used — `ValidationException::withMessages` with `trans()` for plan/ownership violations, silent early-return no-op for state-based ineligibility.
- Create Actions wrap the write in `DB::transaction` with a typed closure returning the model, and assign the UUID via `(string) Str::uuid()` inside it.
- All attributes are assigned explicitly from the typed Data object — no `fill()`, `create()`, `update([...])`, or mass assignment.
- Exactly one Rule class per model, with static, pure, side-effect-free methods.
- Services are free of business-use-case mutations; any Service coordinating multiple Actions is staged through Pipelines (→ labrodev-pipeline).
- Payloads are data-only — public flow-state properties plus a `make()` factory, no behavior, no anonymous arrays between steps (→ labrodev-pipeline).

Full anatomy and templates → labrodev-action skill.
