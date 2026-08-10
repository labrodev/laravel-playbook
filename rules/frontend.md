---
paths:
  - resources/js/**
---

# Frontend (Inertia/React) — playbook checks

- Page file path exactly mirrors the `Inertia::render` string (`pages/{render-path}.tsx`, kebab-case), with a default-exported PascalCase `{Resource}{Action}` component.
- An explicit `interface Props` matches the ViewModel output, with no `translations` prop; enum fields follow the enum contract (render `*_label`, raw value for logic → labrodev-enum).
- All user-facing strings resolve via `t('Readable English')` — zero hardcoded JSX copy, zero dotted slug keys, zero shipped fallbacks.
- All links, form targets, and JSON calls use Wayfinder-generated route functions — no hand-written URLs; `{model:uuid}` values are passed as `item.uuid`.
- Forms use `useForm` with snake_case fields matching the Data class, inline `InputError` display, `processing`-disabled submit, and `FlashAlert` rendered.
- Initial page data is delivered only via Inertia props — no `fetch`/`axios`/`useEffect` loaders; select options come from ViewModel props.
- Destructive actions use an AlertDialog with typed-business-key confirmation and `router.delete()` — no `window.confirm`.
- JSON API calls go through `@/lib/http` with explicit 419/401/403/422 handling and no silently swallowed errors.
- The component is free of business logic, authorization checks, and frontend workarounds for missing backend functionality.
- Inspecting the `data-page` attribute shows only fields the page actually renders — no internal ids or sensitive data.

Full anatomy and templates → labrodev-inertia-react skill.
