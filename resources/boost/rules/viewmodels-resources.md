---
paths:
  - app/Layer/**/ViewModels/**
  - "**/Resources/**"
---

# ViewModels & Resources — playbook checks

- Universal: file opens with `declare(strict_types=1)` and a namespace matching its folder; the class is `final`; names are singular.
- ViewModel extends `Spatie\ViewModels\ViewModel` and overrides `toArray()` returning `array<string, mixed>`.
- ViewModel inputs are constructor-injected as `private readonly`; no DB queries in the ViewModel (relation `load()` in show/edit ViewModels excepted).
- Every model is materialized through a Resource with `->resolve()` — no `JsonResource` instances and no raw models in props.
- Paginated props use the canonical envelope (`data`, `current_page`, `last_page`, `per_page`, `total`, `from`, `to`, `links`).
- Resource is `final` in `App/Layer/{Layer}/{Domain}/Resources` (never under `Core/Domain`), with the `JsonResource<Model>` docblock and the `fetchModel()` guard throwing `ObjectMissed`.
- Resource exposes `uuid` (never the internal integer `id`) and only fields the page actually renders.
- Datetime fields use `->toDateTimeString()` / `->toDateString()`; enum fields emit `value` + `*_label`.
- Nested/eager-loaded relations also pass through allowlisting Resources.
- Output is free of secrets, tokens, password hashes, internal flags, and pivot/debug metadata.
- No business logic, mutation, or Action/Job dispatch in either class.

Full anatomy and templates → labrodev-viewmodel-resource skill.
