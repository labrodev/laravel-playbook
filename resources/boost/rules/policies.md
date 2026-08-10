---
paths:
  - "**/Policies/**"
---

# Policies & Authorization — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular.
- Every permission constant's VALUE exactly equals a policy method name — no dotted strings as constant values.
- The policy is `final`, lives in `Core/Domain/{Domain}/Policies/`, and is named `{Model}Policy`.
- The policy is attached via `#[UsePolicy(...)]` on the model — no `Gate::policy()` call anywhere in a provider.
- Every invokable controller (including JsonControllers) carries class-level `#[Authorize(...)]`, or a documented runtime-setup `Gate::authorize` in the body.
- Instance-ability attributes use the route-parameter name string that matches the `{model:uuid}` route segment.
- Update/remove methods layer checks as: non-null user → ownership/scope → `{Model}Rule` gate.
- Object-state gates are delegated to `{Model}Rule::...` (called statically, not via the model), never re-implemented in the policy.
- The policy is free of mutations, workflows, and calls to Actions/Jobs.
- Only the permissions that real use cases need are defined — no reflexive view/create/update/remove boilerplate.

Full anatomy and templates → labrodev-authorization skill.
